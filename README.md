# Speed-Teste

```javascript

# Autor: HENRIQUE ROSA
# Versão 05
# Descrição: Este script monitora a velocidade de download, intensidade do sinal da rede e qualidade da conexão Ethernet.
# Os dados são enviados para o ThingBoard para monitoramento remoto.

import subprocess
import speedtest
import time
import requests
import json
import psutil
import socket

# Defina o limite mínimo de velocidade desejado em Mbps
MIN_SPEED_THRESHOLD = 25

# Intervalo de verificação em segundos (1 minuto)
CHECK_INTERVAL_SECONDS = 60

# Número de medições para calcular a média móvel
NUM_MEASUREMENTS = 5

# Configurações do ThingBoard
THINGBOARD_URL = "https://thingsboard.cloud"
DEVICE_ACCESS_TOKEN = "EUNUTiDAx3L1N9iNNPMp"
THINGBOARD_API_URL = f"{THINGBOARD_URL}/api/v1/{DEVICE_ACCESS_TOKEN}/telemetry"

# Função para testar a velocidade de download
def test_speed():
    try:
        st = speedtest.Speedtest()
        st.get_best_server()
        download_speed = st.download() / 1024 / 1024  # Convertendo bytes para megabytes
        return download_speed
    except speedtest.ConfigRetrievalError as e:
        print("Erro ao recuperar configurações:", e)
        return None

# Função para calcular a média móvel da velocidade de download
def calculate_moving_average(speed_list, new_speed):
    speed_list.append(new_speed)
    if len(speed_list) > NUM_MEASUREMENTS:
        speed_list.pop(0)
    return sum(speed_list) / len(speed_list)

# Função para obter a largura de banda via speedtest-cli
def get_bandwidth():
    try:
        st = speedtest.Speedtest()
        st.get_best_server()
        download_speed = st.download() / 1024 / 1024  # Convertendo bytes para megabytes
        return download_speed
    except speedtest.ConfigRetrievalError as e:
        print("Erro ao recuperar configurações:", e)
        return None

# Função para classificar a qualidade do sinal
def classify_signal_quality(bandwidth, speed):
    if bandwidth is not None and speed is not None:
        if bandwidth >= MIN_SPEED_THRESHOLD and speed >= MIN_SPEED_THRESHOLD:
            return "Bom"
        else:
            return "Ruim"
    else:
        return "Indisponível"

# Função para verificar a conexão com a internet
def check_internet_connection():
    try:
        socket.create_connection(("www.google.com", 80))
        return True
    except OSError:
        return False

# Função para enviar dados para o ThingBoard
def send_data_to_thingboard(current_speed, moving_average_speed, bandwidth, signal_quality, alert=None, offline_duration=None):
    payload = {
        "current_speed": current_speed,
        "moving_average_speed": moving_average_speed,
        "bandwidth": bandwidth,
        "signal_quality": signal_quality
    }

    if alert:
        payload["alert"] = alert

    if offline_duration is not None:
        payload["offline_duration"] = offline_duration

    headers = {"Content-Type": "application/json"}

    try:
        response = requests.post(THINGBOARD_API_URL, data=json.dumps(payload), headers=headers)
        response.raise_for_status()
        print("Dados enviados para o ThingBoard com sucesso!")
    except requests.exceptions.RequestException as e:
        print("Erro ao enviar dados para o ThingBoard:", e)

if __name__ == "__main__":
    moving_average_speeds = []  # Lista para manter as velocidades para o cálculo da média móvel
    offline_time_start = None  # Armazena o tempo em que a conexão caiu
    offline_duration = 0  # Armazena a duração da conexão offline

    print("         TESTE DE VELOCIDADE         ")
    print("------------------------------------")

    while True:
        current_speed = test_speed()
        bandwidth = get_bandwidth()

        if not check_internet_connection():
            if offline_time_start is None:
                offline_time_start = time.time()
                offline_duration = 0  # Zera a duração da conexão offline quando detectada
        else:
            if offline_time_start is not None:
                offline_duration += time.time() - offline_time_start
                offline_time_start = None

        if current_speed is not None:
            moving_average_speed = calculate_moving_average(moving_average_speeds, current_speed)

            signal_quality = classify_signal_quality(bandwidth, current_speed)

            print(f"Velocidade de download atual: {current_speed:.2f} Mbps")
            print(f"Média móvel de velocidade: {moving_average_speed:.2f} Mbps")

            if bandwidth is not None:
                print(f"Largura de banda: {bandwidth:.2f} Mbps")

            print(f"Qualidade do sinal: {signal_quality}")

            if offline_duration > 0:
                print(f"Tempo sem internet: {offline_duration:.2f} segundos")
                send_data_to_thingboard(current_speed, moving_average_speed, bandwidth, signal_quality, "Sem conexão de internet", offline_duration)
                offline_duration = 0
            else:
                send_data_to_thingboard(current_speed, moving_average_speed, bandwidth, signal_quality)
        else:
            print("Não foi possível realizar o teste de velocidade.")

        time.sleep(CHECK_INTERVAL_SECONDS)
```javascript
