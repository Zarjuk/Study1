import aiohttp
import asyncio
import json
import logging
import time
import aiofiles
from aiogram import Bot, Dispatcher, types
from aiogram.filters import Command
from aiogram.types import Message

logging.basicConfig(level=logging.INFO)

TELEGRAM_TOKEN = 'your_telegram_bot_token'
CHAT_ID = 'your_chat_id'

bot = Bot(token=TELEGRAM_TOKEN)
dp = Dispatcher()

COINS = ["BTC-USDT", "ETH-USDT", "BNB-USDT", "XRP-USDT", "ADA-USDT"]
UPDATE_INTERVAL = 60
BINANCE_URL = "https://api.binance.com/api/v3/ticker/price?symbol="
KUCOIN_URL = "https://api.kucoin.com/api/v1/market/allTickers"
MAX_API_RETRIES = 3
MESSAGE_COOLDOWN = 300

last_message_time = {}

async def send_telegram_message(message_text, coin):
    current_time = time.time()
    if coin in last_message_time and (current_time - last_message_time[coin]) < MESSAGE_COOLDOWN:
        return
    try:
        await bot.send_message(chat_id=CHAT_ID, text=message_text)
        logging.info(f"Message sent to Telegram: {message_text}")
        last_message_time[coin] = current_time
    except Exception as e:
        logging.error(f"Error sending message to Telegram: {e}")

async def get_binance_prices(session, coins):
    prices = {}
    for coin in coins:
        binance_symbol = coin.replace("-", "")
        for attempt in range(MAX_API_RETRIES):
            try:
                async with session.get(BINANCE_URL + binance_symbol) as response:
                    if response.status == 200:
                        data = await response.json()
                        prices[coin] = float(data["price"])
                        break
                    elif response.status == 429:
                        logging.warning("Binance rate limit hit, waiting before retrying...")
                        await asyncio.sleep(60)
                    else:
                        logging.warning(f"Failed to fetch Binance data for {coin}. Status: {response.status}")
            except Exception as e:
                logging.error(f"Error fetching data for {coin} from Binance: {e}")
                await asyncio.sleep(2 ** attempt)
    return prices

async def get_kucoin_prices(session, coins):
    kucoin_prices = {}
    for attempt in range(MAX_API_RETRIES):
        try:
            async with session.get(KUCOIN_URL) as response:
                if response.status == 200:
                    data = await response.json()
                    for token in data["data"]["ticker"]:
                        symbol = token["symbol"]
                        price = float(token["last"])
                        if symbol in coins:
                            kucoin_prices[symbol] = price
                    break
                elif response.status == 429:
                    logging.warning("KuCoin rate limit hit, waiting before retrying...")
                    await asyncio.sleep(60)
                else:
                    logging.warning(f"Failed to fetch Kucoin data. Status: {response.status}")
        except Exception as e:
            logging.error(f"Error fetching data from Kucoin: {e}")
            await asyncio.sleep(2 ** attempt)
    return kucoin_prices

async def compare_prices(binance_prices, kucoin_prices, threshold):
    found_arbitrage = False
    for coin in binance_prices:
        binance_price = binance_prices[coin]
        kucoin_price = kucoin_prices.get(coin)

        if kucoin_price is None:
            logging.warning(f"Coin {coin} not found on Kucoin")
            continue

        price_difference = abs(binance_price - kucoin_price) / ((binance_price + kucoin_price) / 2) * 100

        if price_difference >= threshold:
            found_arbitrage = True
            if binance_price < kucoin_price:
                buy_price, sell_price = binance_price, kucoin_price
                exchange_buy, exchange_sell = "Binance", "Kucoin"
            else:
                buy_price, sell_price = kucoin_price, binance_price
                exchange_buy, exchange_sell = "Kucoin", "Binance"

            profit = sell_price - buy_price
            timestamp = time.strftime("%Y-%m-%d %H:%M:%S", time.gmtime())
            message = (f"Arbitrage opportunity for {coin.replace('-', '')} from {exchange_buy} to {exchange_sell}.\n"
                       f"Buy at: {buy_price}$\n"
                       f"Sell at: {sell_price}$\n"
                       f"Profit: {profit:.5f}$\n"
                       f"Found at: {timestamp}")
            logging.info(message)
            await send_telegram_message(message, coin)

            arbitrage_data = {
                "coin": coin,
                "exchange_buy": exchange_buy,
                "buy_price": buy_price,
                "exchange_sell": exchange_sell,
                "sell_price": sell_price,
                "profit": profit,
                "spread_percentage": price_difference,
                "timestamp": timestamp
            }

            async with aiofiles.open("arbitrage_spreads.json", "a") as f:
                await f.write(json.dumps(arbitrage_data) + "\n")

            print(f"Arbitrage found for {coin} in {time.time() - start_time:.2f} seconds")

    if not found_arbitrage:
        logging.info("No arbitrage found. Continuing...")

async def main(spread_threshold):
    global start_time
    async with aiohttp.ClientSession() as session:
        while True:
            start_time = time.time()
            logging.info("Fetching prices from Binance and Kucoin...")
            binance_prices = await get_binance_prices(session, COINS)
            kucoin_prices = await get_kucoin_prices(session, COINS)
            await compare_prices(binance_prices, kucoin_prices, spread_threshold)
            logging.info(f"Waiting {UPDATE_INTERVAL} seconds before next update...")
            await asyncio.sleep(UPDATE_INTERVAL)

@dp.message(Command(commands=['start']))
async def start(message: Message):
    await message.answer("Hello! I'm your arbitrage bot, monitoring Binance and Kucoin prices.")
    logging.info("Bot started.")

@dp.message(Command(commands=['setthreshold']))
async def set_threshold(message: Message):
    global spread_threshold
    try:
        threshold_value = float(message.text.split()[1])
        spread_threshold = threshold_value
        await message.answer(f"Spread threshold set to {spread_threshold}%")
        logging.info(f"Spread threshold set to: {spread_threshold}%")
    except (IndexError, ValueError):
        await message.answer("Please enter a valid number for the threshold.")

async def telegram_polling():
    await dp.start_polling(bot)

async def run_bot(spread_threshold):
    main_task = asyncio.create_task(main(spread_threshold))
    polling_task = asyncio.create_task(telegram_polling())
    try:
        await asyncio.gather(main_task, polling_task)
    except asyncio.CancelledError:
        logging.info("Bot shutting down...")
    finally:
        main_task.cancel()
        polling_task.cancel()
        await main_task
        await polling_task

if __name__ == "__main__":
    spread_threshold_input = input("Enter spread threshold (default 2%): ")
    spread_threshold = float(spread_threshold_input.replace(',', '.')) if spread_threshold_input else 2.0

    try:
        asyncio.run(run_bot(spread_threshold))
    except KeyboardInterrupt:
        logging.info("Program interrupted and shutting down...")

