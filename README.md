# bot-binance
import pandas as pd
import ta
import time
import requests
from binance.client import Client
from binance.enums import *
from datetime import datetime
import os

# --- CONFIGURACIÓN TELEGRAM ---
TELEGRAM_BOT_TOKEN = '7752503466:AAHX-aaTdhmrKT-qd739cSzu4mpCQ-E5gvQ'
TELEGRAM_CHAT_ID = '6193076225'

# --- CLAVES TESTNET SPOT ---
API_KEY = '6lg21fD6eO4aAItM1PI45nH1n6ulVrrSyEUGWrpv03bMWioxbwbi68mEiF2UyuQq'
API_SECRET = 'ynmnZ066o2yRDxAQWjXVgEIAAT5id5QnBZanGFGvmd4hLN49eQwEVeFbGR4VewfZ'

# Cliente con testnet habilitado
client = Client(API_KEY, API_SECRET)
client.API_URL = 'https://testnet.binance.vision/api'

# --- CONFIGURACIÓN ---
SIMBOLOS = ['BTCUSDT', 'ETHUSDT', 'BNBUSDT']
CANTIDAD = 0.001  # Monto de prueba (ajústalo según el par)
CSV_FILENAME = "historial_testnet.csv"

def enviar_telegram(mensaje):
    url = f'https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage'
    requests.post(url, data={'chat_id': TELEGRAM_CHAT_ID, 'text': mensaje})

def obtener_datos(simbolo):
    klines = client.get_klines(symbol=simbolo, interval=Client.KLINE_INTERVAL_1MINUTE, limit=50)
    df = pd.DataFrame(klines, columns=[
        'timestamp', 'Open', 'High', 'Low', 'Close', 'Volume', 'Close_time',
        'Quote_asset_volume', 'Number_of_trades', 'Taker_buy_base', 'Taker_buy_quote', 'Ignore'
    ])
    df['Close'] = df['Close'].astype(float)
    df.set_index(pd.to_datetime(df['timestamp'], unit='ms'), inplace=True)
    return df

def detectar_senal(df):
    df['rsi'] = ta.momentum.RSIIndicator(df['Close']).rsi()
    macd = ta.trend.MACD(df['Close'])
    df['macd'] = macd.macd()
    df['macd_signal'] = macd.macd_signal()

    ultima = df.iloc[-1]
    if ultima['rsi'] < 35 and ultima['macd'] > ultima['macd_signal']:
        return "BUY"
    elif ultima['rsi'] > 65 and ultima['macd'] < ultima['macd_signal']:
        return "SELL"
    return "NONE"

def ejecutar_operacion(simbolo, tipo, precio):
    try:
        if tipo == "BUY":
            client.order_market_buy(symbol=simbolo, quantity=CANTIDAD)
        elif tipo == "SELL":
            client.order_market_sell(symbol=simbolo, quantity=CANTIDAD)

        mensaje = f"🧪 ORDEN TESTNET {tipo} en {simbolo}\n💰 Precio aprox: {precio:.2f}"
        print(mensaje)
        enviar_telegram(mensaje)

        fila = {
            "timestamp": datetime.now().strftime("%Y-%m-%d %H:%M"),
            "simbolo": simbolo,
            "tipo": tipo,
            "precio": precio,
            "cantidad": CANTIDAD
        }
        pd.DataFrame([fila]).to_csv(CSV_FILENAME, mode='a', header=not os.path.exists(CSV_FILENAME), index=False)
    except Exception as e:
        print(f"❌ Error en orden testnet: {e}")
        enviar_telegram(f"❌ Error en orden testnet: {e}")

def main():
    while True:
        for simbolo in SIMBOLOS:
            try:
                df = obtener_datos(simbolo)
                senal = detectar_senal(df)
                precio = df['Close'].iloc[-1]

                if senal != "NONE":
                    ejecutar_operacion(simbolo, senal, precio)
                else:
                    print(f"⏳ [{simbolo}] Sin señal válida. Precio: {precio:.2f}")
            except Exception as e:
                print(f"⚠️ Error con {simbolo}: {e}")
                enviar_telegram(f"⚠️ Error con {simbolo}: {e}")
        time.sleep(60)

if __name__ == "__main__":
    main()
