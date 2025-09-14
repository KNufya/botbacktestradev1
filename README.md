# botbacktestradev1
"""
SimpleFX ORB Bot – Step 2 (WS Trading)

Este script:
1. Hace login vía REST y obtiene el access_token.
2. Construye la URL WebSocket con el token.
3. Se conecta al WS.
4. Envía una orden LIMIT BUY en US100 (cuenta DEMO 3001154).

Dependencias: requests, websocket-client
"""

import os
import json
import uuid
import requests
import websocket

BASE_URL = "https://rest.simplefx.com/api/v3"
AUTH_ENDPOINT = "/auth/key"

ACCOUNT_ID = "3001154_DEMO"   # tu cuenta DEMO en USD
SYMBOL = "US100"


def get_access_token(client_id: str, client_secret: str) -> str:
    """Autenticación REST para obtener access_token."""
    url = BASE_URL + AUTH_ENDPOINT
    body = {"clientId": client_id, "clientSecret": client_secret}

    r = requests.post(
        url,
        json=body,
        headers={"Accept": "application/json", "Content-Type": "application/json"},
        timeout=20,
    )
    if r.status_code != 200:
        raise RuntimeError(f"Auth fallida {r.status_code}: {r.text}")

    token = r.json().get("data", {}).get("token")
    if not token:
        raise RuntimeError(f"Auth fallida, no token en respuesta: {r.text}")
    return token


def on_open(ws):
    print("✔ WebSocket abierto")

    # generar IDs únicos
    req_id = f"bot-{uuid.uuid4()}"

    order_msg = {
        "p": "/order/open",
        "d": {
            "accountId": ACCOUNT_ID,
            "symbol": SYMBOL,
            "action": "buy",
            "volume": 0.01,
            "type": "limit",
            "price": 5000,  # alejado para que quede pendiente
            "timeInForce": "GTC"
        },
        "webRequestId": req_id,
        "clientRequestId": req_id
    }

    ws.send(json.dumps(order_msg))
    print("→ Enviada orden LIMIT BUY US100 (pendiente)")


def on_message(ws, message):
    print("Mensaje recibido:", message)


def on_error(ws, error):
    print("Error WS:", error)


def on_close(ws, code, msg):
    print("WebSocket cerrado:", code, msg)


if __name__ == "__main__":
    client_id = os.getenv("SIMPLEFX_CLIENT_ID")
    client_secret = os.getenv("SIMPLEFX_CLIENT_SECRET")
    if not client_id or not client_secret:
        print("Faltan SIMPLEFX_CLIENT_ID o SIMPLEFX_CLIENT_SECRET")
        exit(1)

    print("[SimpleFX] WS Test: Orden LIMIT en US100 (DEMO)")
    token = get_access_token(client_id, client_secret)
    print("✔ Token obtenido")

    # Construir URL WS con el token
    ws_url = (
        f"wss://rest.simplefx.com/api/v3/websocket"
        f"?application=desktop&application_version=2.1.346.11"
        f"&access_token={token}"
    )
    print("→ Conectando a:", ws_url)

    ws = websocket.WebSocketApp(
        ws_url,
        on_open=on_open,
        on_message=on_message,
        on_error=on_error,
        on_close=on_close,
    )
    ws.run_forever()
