# Read skylanders portal data

# **Objectif du projet**

L'objectif est de détecter la présence ou non d'une figurine Skylanders sur le portail avec un script Python et de déclencher des actions en réponse.

Les figurines sont dotées de puces NFC qui contiennent des informations. Le portail agit comme un lecteur qui émet des ondes électromagnétiques. En les faisant se rapprocher, nous allons les faire entrer en communication. La figurine va alors lui transmettre ses données. Habituellement, celles-ci sont ensuite envoyées à la console via le port USB, mais à la place, nous utiliserons un script Python qui les récupèrera sur notre ordinateur.

# I- **Conception logicielle**

## a) PyUSB

Dans un premier temps, nous devons installer la bibliothèque PyUSB qui va nous permettre d'interagir avec nos dispositifs USB sur Python. Notre script utilisera cette bibliothèque pour faciliter la communication avec le portail.

```python
pip install pyusb
```

PyUSB est une bibliothèque qui sert d'interface entre le script Python et libusb.

## b) LibUSB

Libusb est une bibliothèque qui fournit un accès générique aux dispositifs USB. Elle permet aux applications qui ont besoin de communiquer avec des dispositifs USB (notre script) de ne pas avoir besoin de leurs pilotes spécifiques.

Cette bibliothèque traite les commandes de PyUSB et les transmet au pilote USB utilisé, dans notre cas libusb-win32.

## c) **Pilote libusb-win32**

Libusb-win32 est un pilote USB qui permet à libusb de communiquer avec les dispositifs USB. Nous allons donc pouvoir lire des données sur le portail sans dépendre des pilotes spécifiques développés par le fabricant pour le jeu.

Pour son installation, nous utiliserons le logiciel Zadig. Sur l'interface, nous sélectionnons le bon périphérique du portail, Spyro, et installons le pilote libusb-win32.

![Screenshot_(848)](https://github.com/Haki-i/read-skylanders-portal-data/assets/137703849/fb3ce498-5b34-4aef-8605-222e680b4ba6)

- Avec Zadig, nous installons le pilote libusb-win32 sur le portail, ce qui nous permet de gérer les communications USB entre le portail et l'ordinateur.
- Ensuite, lorsque nous exécutons un script Python qui utilise PyUSB, cette bibliothèque va appeler libusb pour lire des données sur le portail.
- Enfin, libusb interagit avec le portail Skylanders en utilisant le pilote libusb-win32 que nous avons installé.

# II- **Conception informatique**

## a) Informations du portail

Pour commencer, nous devons importer la bibliothèque PyUSB.

```python
import usb.core
import usb.util
```

En appelant la méthode **`find`**, nous affichons tous les dispositifs USB connectés, y compris notre portail.

```python
dispositifs = usb.core.find(find_all=True)
for dispositif in dispositifs:
    print('Dispositif trouvé:', dispositif)
```

![9947ac8b-ca88-46a5-badb-818a7fdd8b7b](https://github.com/Haki-i/read-skylanders-portal-data/assets/137703849/139d5fa4-961c-4efb-8514-4ed1a65b47ce)

Dans l'ensemble des informations reçues, nous obtenons deux codes hexadécimaux qui correspondent à l'ID du vendeur et à l'ID du produit (le portail). Ces identifiants nous permettront d'identifier notre portail par la suite.

![239ec608-f1d9-4a00-a85b-dc0e04085687](https://github.com/Haki-i/read-skylanders-portal-data/assets/137703849/566913c2-2b2e-4147-92ab-52b84890901e)

Nous pouvons également observer plusieurs interfaces qui contiennent chacune différents endpoints. Ces endpoints sont les canaux par lesquels les données sont reçues par l'ordinateur (IN) ou envoyées vers le portail (OUT).

De manière empirique, nous trouvons que l'endpoint d'interruption IN ayant l'adresse 0x81 dans l'interface 0 va nous permettre de détecter si une figurine est posée.

Avec ces données, nous pouvons refaire une recherche des dispositifs USB connectés en spécifiant l'ID du vendeur et l'ID du produit pour être certain de sélectionner notre portail.

```python
dispositif = usb.core.find(idVendor=ID_VENDEUR, idProduct=ID_PRODUIT)
```

Nous déclarons ensuite l'adresse de l'endpoint et son interface, puis nous utilisons la méthode `claim_interface`. Cette méthode permet à notre script d'avoir un accès exclusif à cette interface du dispositif USB, dans le cas où d'autres processus pourraient tenter d'y accéder simultanément.

```python
endpoint_in = 0x81
interface = 0
usb.util.claim_interface(dispositif, interface)
```

## b) Lecture des données

Pour lire les 32 bits de données envoyés par le portail à cet endpoint, nous utiliserons la méthode **`read`** :

```python
data = dispositif.read(endpoint_in | usb.util.ENDPOINT_IN, 32, timeout=500)
```

Nous obtenons alors un tableau dont la plupart des valeurs sont fixes, à l'exception de ce qui semble être un compteur à l'indice 7.

Nous constatons également que lorsque nous posons une figurine sur le portail, un changement de valeur s'effectue à l'indice 3.

![014a8004-2959-40a5-8372-344a8d02a289](https://github.com/Haki-i/read-skylanders-portal-data/assets/137703849/32003314-d828-4209-b2b7-ebdc43474d50)

Nous comprenons donc que :

- Les valeurs 0 et 1 représentent des états constants qui indiquent respectivement qu'aucune figurine n’est posée et qu'une figurine est posée sur le portail.
- Les valeurs 2 et 3 représentent des états transitoires pour indiquer que la figurine vient d'être retirée ou vient d'être posée.

Grâce à cela, nous pouvons mettre en place différentes conditions pour déclencher divers événements.

```python
if data[3] == 0:
     print("Aucune figurine")
elif data[3] == 3:
     print("figurine posée")
elif data[3] == 1:
     print("figurine sur le portail")
elif data[3] == 2:
     print("figurine retirée du portail")
```
![20240508_144628-ezgif com-optimize](https://github.com/Haki-i/read-skylanders-portal-data/assets/137703849/ea90be5e-e47c-4da2-a5a1-718e572fe785)


