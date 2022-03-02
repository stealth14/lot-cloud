# lot-cloud

[Demostración en video](https://youtu.be/kMU-U_RUYlw).

## Desarrollo de la práctica 

Instalación de python 3

![Picture1](https://user-images.githubusercontent.com/52419137/156303945-3aff548a-3048-48a9-ae26-2b067ba892b8.png)

Creación de proyecto firebase

![Picture2](https://user-images.githubusercontent.com/52419137/156304186-fdc15988-fab7-444c-ab4f-7bc3b3fe15bd.png)

Habilitar el servicio firestore

![Picture3](https://user-images.githubusercontent.com/52419137/156304199-5979f9d2-6693-4852-8982-2e92e2560c92.png)

Habilitar método de autenticación anónimo
 
![Picture4](https://user-images.githubusercontent.com/52419137/156304211-23ea37cc-0aa2-42ca-a409-07acf2b384d8.png)


Instalar las herramientas de desarrollador python de google cloud firestore admin sdk

```pip install --upgrade firebase-admin```

![Picture5](https://user-images.githubusercontent.com/52419137/156304232-41245633-a0d3-4da8-a715-ad232d8f85cb.png)

Generar archivo clave desde la sección de cuentas de servicio correspondiente al proyecto, en este caso “iot-clou”

![Picture6](https://user-images.githubusercontent.com/52419137/156304442-859c7e8e-1946-4b6e-ad23-0bcf76cb94ae.png)

Después de generar la clave se descargará un archivo json.


![Picture7](https://user-images.githubusercontent.com/52419137/156304491-67d0bbeb-520f-4af9-909f-14927dd5a015.png)

 
Copiar este archivo en cualquier directorio de la maquina virtual.

![Picture8](https://user-images.githubusercontent.com/52419137/156304516-6934840b-8c08-4db0-a7d8-c7d5744c8266.png)

En este caso generamos las credenciales necesarias en la línea 3 del script, misma en la que referenciamos el archivo json que fue copiado en el paso anterior.

![Picture9](https://user-images.githubusercontent.com/52419137/156304552-fb0090d4-af08-4c8b-a093-e94ed7c87374.png)

Instalar sdk cliente para Python de thingsboard 

![Picture10](https://user-images.githubusercontent.com/52419137/156304566-97af2e5c-e412-4211-a8d4-a339829cd1fd.png)

```pip3 install tb-mqtt-client```

Crear dispositivo de thingsboard

![Screen Shot 2022-03-02 at 01 08 53](https://user-images.githubusercontent.com/52419137/156305288-845a7511-6d8b-4043-8d9c-4159426d89a4.png)

Crear Dashboard en thingsboard

![Screen Shot 2022-03-02 at 01 09 11](https://user-images.githubusercontent.com/52419137/156305306-946c63a9-857b-4d54-9dbc-8bef9566f1c9.png)

Se añade la funcion que permite enviar los datos de temperatura al dashboard configurado en el paso anterior

![Picture11](https://user-images.githubusercontent.com/52419137/156304585-7a88e86e-11ac-4cf0-b15f-bc98cb5474ed.png)

Script completo de python ejecutado en Raspberry Pi OS :

```
import firebase_admin
from firebase_admin import credentials
from firebase_admin import firestore
    
#inicia cliente de firebase
    
cred = credentials.Certificate('/home/pi/Desktop/iot-clou-9b1529fca1a9.json')
firebase_admin.initialize_app(cred)

db = firestore.client()

#inicia emulador de sensor de temperatura

from sense_emu import SenseHat

sense = SenseHat()

# clase para crear intervalo

from threading import Timer

class Repeat(Timer):
    def run(self):
        while not self.finished.wait(self.interval):
            self.function(*self.args, **self.kwargs)
            
from time import sleep
import datetime

# funcion a ejecutar en el intervalo

def sendToFirebase():
    # leer la temperatura
    temp = sense.temp

    # enviar a firestore
    doc_ref = db.collection(u'temperatures').document()

    doc_ref.set({
    u'value': temp,
    u'date': datetime.datetime.now(),
    })

# enviar a thingsboard
from tb_device_mqtt import TBDeviceMqttClient, TBPublishInfo

client = TBDeviceMqttClient("thingsboard.cloud", "nyiGgJTpTCuJB9hD1vjY")
# Connect to ThingsBoard

client.connect()

def sendToThingsboard():
    temp = sense.temp

    telemetry = {"temperature": temp, "enabled": False, "currentFirmwareVersion": "v1.2.2"}
    
    # Sending telemetry without checking the delivery status
    client.send_telemetry(telemetry) 
    # Sending telemetry and checking the delivery status (QoS = 1 by default)
    result = client.send_telemetry(telemetry)
    # get is a blocking call that awaits delivery status  
    success = result.get() == TBPublishInfo.TB_ERR_SUCCESS


def capture():
    sendToFirebase()
    sendToThingsboard()
    print("...")


# capturar lecturas de temperatura durante 10 segundos

print("Enviando temperatura: Sensor 1...")

t = Repeat(2.0, lambda:capture() )
t.start()
sleep(60)
t.cancel()

# Disconnect from ThingsBoard
client.disconnect()




```
