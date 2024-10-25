import aiohttp
import asyncio
import json
import time
import logging
import argparse

logging.basicConfig(filename="okx_parser.log", level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")


class OKXAPI:
    BASE_URL = 'https://www.okx.com'

    async def fetch(self, session, endpoint: str):
    
        url = f"{self.BASE_URL}{endpoint}"
        try:
            async with session.get(url) as response:
                return await response.json()
        except aiohttp.ClientConnectionError as e:
            logging.error(f"Ошибка подключения: {e}")
            print(f"Ошибка подключения: {e}")
        except aiohttp.ServerTimeoutError as e:
            logging.error(f"Таймаут соединения с сервером: {e}")
            print(f"Таймаут соединения с сервером: {e}")
        except Exception as e:
            logging.error(f"Произошла непредвиденная ошибка: {e}")
            print(f"Произошла непредвиденная ошибка: {e}")
        return None

    async def get_all_coins(self, session):
       
        endpoint = "/api/v5/market/tickers?instType=SPOT"
        data = await self.fetch(session, endpoint)
        return data['data'] if data else []


class CoinDataParser:
    def __init__(self, api_client):
        self.api_client = api_client

    async def fetch_coin_prices(self, session):
     
        coins_data = await self.api_client.get_all_coins(session)
        return [
            {
                'symbol': coin['instId'],
                'last_price': coin['last'],
                'base_currency': coin['instId'].split('-')[0],
                'quote_currency': coin['instId'].split('-')[1]
            }
            for coin in coins_data
        ]


async def save_json(data, filename='coin_prices.json'):
 
    try:
        with open(filename, 'w') as json_file:
            json.dump(data, json_file, indent=4)
        logging.info(f"Данные успешно сохранены в файл {filename}.")
        print(f"Данные успешно сохранены в файл {filename}.")
    except Exception as e:
        logging.error(f"Ошибка при сохранении данных в файл: {e}")
        print(f"Ошибка при сохранении данных в файл: {e}")


def parse_args():
 
    parser = argparse.ArgumentParser(description="OKX Data Parser")
    parser.add_argument("--filename", default="coin_prices.json", help="Output filename for the results")
    return parser.parse_args()


async def main(filename):
    start_time = time.time()
    print("Начало работы программы...")
    logging.info("Начало работы программы...")

    try:
        api = OKXAPI()
        parser = CoinDataParser(api)

        async with aiohttp.ClientSession() as session:
            coin_prices = await parser.fetch_coin_prices(session)

            if coin_prices:
                print(f"Найдена информация по {len(coin_prices)} валютным парам.")
                logging.info(f"Найдена информация по {len(coin_prices)} валютным парам.")
            else:
                print("Не удалось получить информацию о валютных парах.")
                logging.warning("Не удалось получить информацию о валютных парах.")

            await save_json(coin_prices, filename)
    except Exception as e:
        logging.error(f"Произошла непредвиденная ошибка: {e}")
        print(f"Произошла непредвиденная ошибка: {e}")
    finally:
        end_time = time.time()
        execution_time = end_time - start_time
        print(f"Работа программы завершена за {execution_time:.2f} секунд.")
        logging.info(f"Работа программы завершена за {execution_time:.2f} секунд.")


if __name__ == "__main__":
    args = parse_args()
    asyncio.run(main(args.filename))
