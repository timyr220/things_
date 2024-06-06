# things
import requests

def authenticate(url, username, password):
    auth_url = f"{url}/api/auth/login"
    payload = {
        "username": username,
        "password": password
    }
    response = requests.post(auth_url, json=payload)
    if response.status_code == 200:
        return response.json()['token']
    else:
        print("Authentication failed:", response.status_code, response.json())
        return None


# Сбор телеметрии с PE
def get_telemetry(token, url, device_id):
    telemetry_url = f"{url}/api/plugins/telemetry/DEVICE/{device_id}/values/timeseries"
    headers = {
        'accept': 'application/json',
        'X-Authorization': f'Bearer {token}'
    }
    response = requests.get(telemetry_url, headers=headers)
    if response.status_code == 200:
        return response.json()
    else:
        print("Failed to get telemetry:", response.status_code, response.json())
        return None


# Отправка телеметрии на CE
def send_telemetry(url, device_id, telemetry_data, token):
    telemetry_url = f"{url}/api/plugins/telemetry/DEVICE/{device_id}/timeseries/ANY?scope=ANY"
    headers = {
        'Content-Type': 'application/json',
        'X-Authorization': f'Bearer {token}'
    }
    response = requests.post(telemetry_url, json=telemetry_data, headers=headers)
    try:
        response_json = response.json()
    except ValueError:
        response_json = None

    if response.status_code == 200:
        print("Telemetry sent successfully")
    else:
        print("Failed to send telemetry:", response.status_code, response_json, response.text)


# Основная программа
tb_pe_url = "http://localhost:8080"
tb_ce_url = "http://10.7.2.159:8080"
username = "tenant@thingsboard.org"
password = "tenant"

# Получение токенов аутентификации
pe_token = authenticate(tb_pe_url, username, password)
ce_token = authenticate(tb_ce_url, username, password)

if pe_token and ce_token:
    print(f"PE Token: {pe_token}")
    print(f"CE Token: {ce_token}")

    pe_device_id = "69c0d3e0-1f34-11ef-adea-f366d2a9ddce"
    telemetry_data = get_telemetry(pe_token, tb_pe_url, pe_device_id)

    if telemetry_data:
        print(f"Telemetry Data: {telemetry_data}")

        ce_device_id = "d6367250-1f4d-11ef-8992-094cb3eb5a17"

        telemetry_payload = {
            "ts": telemetry_data["temperature"][0]["ts"],
            "values": {
                "temperature": telemetry_data["temperature"][0]["value"]
            }
        }
        print(f"Telemetry Payload: {telemetry_payload}")
        print(f"Sending telemetry to: {tb_ce_url}/api/plugins/telemetry/DEVICE/{ce_device_id}/timeseries/ANY?scope=ANY")
        send_telemetry(tb_ce_url, ce_device_id, telemetry_payload, ce_token)
else:
    print("Failed to authenticate in ThingsBoard PE or CE.")
