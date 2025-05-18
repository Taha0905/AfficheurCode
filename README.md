# AfficheurCode
import socket
import paho.mqtt.client as mqtt
 
# -----------------------
# Partie TCP Afficheur
# -----------------------
def envoyer_message_afficheur(message):
    HOST = "172.31.253.147"  # IP de ton afficheur
    PORT = 10001             # Port du protocole TCP-ASCII
 
    # Message au format DTPM : Mode immédiat + message + fin de trame (0x0D)
    message_bytes = b'\x04\xF0' + message.encode() + b'\r'
 
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.connect((HOST, PORT))
            s.sendall(message_bytes)
            print(f"Message envoyé à l'afficheur : {message}")
    except Exception as e:
        print(f"Erreur lors de l’envoi à l’afficheur : {e}")
 
# Test initial : envoyer "hello"
envoyer_message_afficheur("hello")
 
# -----------------------
# Partie MQTT
# -----------------------
places_etat = {}
dernier_nombre_libres = -1   # Pour éviter l'envoi si pas de changement
 
def on_connect(client, userdata, flags, rc):
    print(f"Connecté au broker MQTT avec le code {rc}")
    topics = [
        "places/place1", "places/place2", "places/place3",
        "places/place4", "places/place5", "places/place6"
    ]
    for topic in topics:
        places_etat[topic] = "Inconnu"
        client.subscribe(topic)
 
def on_message(client, userdata, msg):
    global dernier_nombre_libres
 
    topic = msg.topic
    payload = msg.payload.decode('utf-8')
    places_etat[topic] = payload
 
    # Compter les places libres
    libres = [t for t, etat in places_etat.items() if etat.lower() == "libre"]
    nb_libres = len(libres)
 
    print(f"Places libres : {nb_libres} | {libres}")
 
    # N’envoyer à l’afficheur QUE si le nombre a changé
    if nb_libres != dernier_nombre_libres:
        envoyer_message_afficheur(f"PLACES LIBRES : {nb_libres}")
        dernier_nombre_libres = nb_libres
 
# Connexion au broker
broker_ip = "172.31.254.254"
broker_port = 1883
 
client = mqtt.Client(client_id="SmartParking_Python")
client.on_connect = on_connect
client.on_message = on_message
 
client.connect(broker_ip, broker_port, 60)
client.loop_forever()
 
 
