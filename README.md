<img width="2048" height="768" alt="image" src="https://github.com/user-attachments/assets/79d1a3a4-1ce1-43ac-9658-2f235947f9ac" />




#  Guide pour trouver ses LocalKey, DeviceID et DPs de ses appareils Tuya wifi — depuis Home Assistant OS - sans devoir utiliser de compte dev sur iot.tuya.com



Guide pour récupérer le `device_id` et la `local_key` d'un appareil Tuya WiFi, l'interroger avec `tinytuya`, puis déduire le nom, le type, la plage et le pas (step) de chaque DP en le comparant à l'app Smart Life — **tout se fait depuis Home Assistant OS**.

## Sommaire

1. [À quoi sert ce guide](#1-à-quoi-sert-ce-guide)
2. [Prérequis](#2-prérequis)
3. [Installer l'add-on Advanced SSH & Web Terminal](#3-installer-laddon-advanced-ssh--web-terminal)
4. [Vérifier / installer tinytuya dans le conteneur Home Assistant](#4-vérifier--installer-tinytuya-dans-le-conteneur-home-assistant)
5. [Récupérer device_id et local_key avec tuya-local-key](#5-récupérer-device_id-et-local_key-avec-tuya-local-key)
6. [Lire les DPs de l'appareil](#6-lire-les-dps-de-lappareil)
7. [Monitoring live des DPs + déduction via l'app Smart Life](#7-monitoring-live-des-dps--déduction-via-lapp-smart-life)
8. [Tableau de suivi](#8-tableau-de-suivi)
9. [Ressources](#9-ressources)

---

## 1. À quoi sert ce guide

<details>
<summary>Voir plus</summary>

Le `device_id` et la `local_key` récupérés ici servent à ajouter manuellement un appareil WiFi Tuya dans Home Assistant via une intégration custom HACS en local — **sans passer par le cloud Tuya à l'usage**, et surtout **sans avoir besoin de créer de compte ni de projet développeur sur `iot.tuya.com`**. Deux intégrations HACS courantes utilisent ces informations :

- **[tuya-local](https://github.com/make-all/tuya-local)** — configs par appareil basées sur des profils YAML communautaires.
- **[localtuya](https://github.com/c/localtuya/)** — configuration manuelle DP par DP, entité par entité.

Le mapping des DPs (nom, type, range, step) construit dans ce guide sert précisément à configurer correctement ces entités dans l'intégration choisie, une fois le `device_id`/`local_key` en main.

> ⚠️ **Si vous utilisez localtuya** : préférez le fork **[xZetsubou/hass-localtuya](https://github.com/xZetsubou/hass-localtuya)** au dépôt original `rospogrigio/localtuya` listé plus haut. Le fork xZetsubou est mieux maintenu et compatible avec le protocole **3.5**, alors que le dépôt original ne l'est pas — un appareil récent utilisant le protocole Tuya 3.5 restera invisible ou impossible à configurer avec la version originale de localtuya (fork rospogrigio).

</details>

---

## 2. Prérequis

<details>
<summary>Voir plus</summary>

- Home Assistant OS (ou Supervised), avec accès au panneau **Paramètres → Add-ons (Applications)**.
- L'appareil Tuya déjà appairé et fonctionnel dans l'app **Smart Life** (ou Tuya Smart) sur un téléphone.
- Le téléphone avec l'app Smartlife à portée, pour scanner un QR code et pour manipuler l'appareil tuya pendant le monitoring.

**Aucun compte développeur sur `iot.tuya.com` n'est nécessaire pour ce guide.**

</details>

---

## 3. Installer l'add-on Advanced SSH & Web Terminal

<details>
<summary>Voir plus</summary>

L'add-on officiel "Terminal & SSH" ne donne pas accès à Docker. Il faut l'add-on communautaire **Advanced SSH & Web Terminal**, qui donne un véritable shell sur l'hôte (avec `docker`).

> ⚠️ Ce guide donne un accès complet à l'hôte Docker de l'installation HA (protection mode désactivé sur l'add-on). C'est nécessaire pour les commandes `docker exec`/`docker run` ci-dessous, mais cela sort du cadre "sécurisé par défaut" de HAOS — restez prudent avec les commandes exécutées, et désactivez/retirez l'add-on une fois le travail terminé, si désiré, pour revenir à un système plus verrouillé.

[![Ouvrir votre Home Assistant et ajouter ce dépôt d'add-ons.](https://my.home-assistant.io/badges/supervisor_add_addon_repository.svg)](https://my.home-assistant.io/redirect/supervisor_add_addon_repository/?repository_url=https%3A%2F%2Fgithub.com%2Fhassio-addons%2Frepository)

1. **Paramètres → Add-ons → Boutique d'add-ons**.
2. Menu **⋮** (en haut à droite) → **Dépôts** → ajouter :
   ```
   https://github.com/hassio-addons/repository
   ```
3. Rechercher **"Advanced SSH & Web Terminal"** dans la liste (section "Home Assistant Community Add-ons") et cliquer sur **Installer**.
4. Une fois installé, se rendre dans l'onglet **Configuration** de l'add-on :
   - Définir un mot de passe (`password`) ou une clé SSH.
   - Mettre **`Protection mode` sur `OFF`** — c'est ce qui donne l'accès à Docker/à l'hôte. Sans cela, les commandes plus bas ne fonctionneront pas.
5. Démarrer l'add-on (onglet **Info** → **Démarrer**), puis activer **"Afficher dans le menu"** pour avoir un accès Terminal directement dans la barre latérale de Home Assistant.
6. Ouvrir le terminal (icône dans la barre latérale, ou via l'onglet **Web UI** de l'add-on).

Un shell avec accès complet à Docker est maintenant disponible, directement depuis la sidebar de Home Assistant

</details>

---

## 4. Vérifier la présence / installer tinytuya dans le conteneur Home Assistant

<details>
<summary>Voir plus</summary>

Dans le terminal de l'add-on :

```bash
docker exec -it homeassistant pip show tinytuya
```

S'il est absent :

```bash
docker exec -it homeassistant pip install tinytuya
```

> ⚠️ Une installation manuelle dans le conteneur ne survit pas à une recréation du conteneur (mise à jour de l'image HA). Pour une utilisation ponctuelle de reverse engineering, ce n'est pas un problème — il suffit de le réinstaller au besoin.

</details>

---

## 5. Récupérer le device_id et local_key avec tuya-local-key

<details>
<summary>Voir plus</summary>

Bien que le device_id soit récupérable directement via l'app Smartlife, la local_keym qui est aussi indispensable,n'est pas exposée dans l'app smartlife.

[`tuya-local-key`](https://github.com/vineetchoudhary/tuya-local-key) récupère la liste des appareils (ID, local key, etc...) via une connexion QR code au compte Smart Life — **sans créer de projet développeur Tuya**. Il existe en version add-on Home Assistant.

> ⚠️ **Attention** L'adresse IP exposée par tuya-local-key correspond à votre adresse IP publique attribué par votre FAI et non à l'adresse IP locale de votre appareil. Cette adresse IP n'est pas celle qui sera utilisé plus tard dans cette procédure.

### Installation (en tant qu'add-on)

[![Ouvrir votre Home Assistant et ajouter ce dépôt d'add-ons.](https://my.home-assistant.io/badges/supervisor_add_addon_repository.svg)](https://my.home-assistant.io/redirect/supervisor_add_addon_repository/?repository_url=https%3A%2F%2Fgithub.com%2Fvineetchoudhary%2Ftuya-local-key)

1. **Paramètres → Add-ons → Boutique d'add-ons**.
2. Menu **⋮** (en haut à droite) → **Dépôts** → ajouter :
   ```
   https://github.com/vineetchoudhary/tuya-local-key
   ```
3. Rechercher **"Tuya Local Key"** dans la liste et cliquer sur **Installer**.
4. Une fois installé, le démarrer, puis l'ouvrir directement depuis la barre latérale de Home Assistant (l'add-on utilise l'ingress de Home Assistant — pas besoin de connaître une IP ou un numéro de port).

### Utilisation

1. Dans l'app Smart Life sur votre téléphone: **Moi → Paramètres → Compte et sécurité → Code utilisateur**. Noter ce code et l'entrer dans l'interface de l'add-on.

<img width="240" height="500" alt="user_code" src="https://github.com/user-attachments/assets/79e53ef0-4aea-4459-b018-92d38d6b7d1a" />


2. Un QR code s'affiche dans l'interface de l'add-on. Le scanner avec l'app Smart Life :

<img width="271" height="559" alt="image" src="https://github.com/user-attachments/assets/b3415450-7116-4aec-ba5f-f5f8071fb5ca" />




3. Une demande de connexion apparaît dans l'app — appuyer sur **"Confirmer la connexion"**.
4. La liste des appareils apparaît dans l'add-on : `device_id`, `local_key`, `uuid`, `catégorie`, IP locale, statut en ligne.
5. Prendre en note les  `device_id`, `local_key`. Ne pas confondre `device_id` (appelé simplement "id" dans cet addon avec le `procduct id`

<img width="1591" height="357" alt="image" src="https://github.com/user-attachments/assets/f6e04bdf-4db6-4241-bb04-04b522a18882" />

> ⚠️ **Note** : Une fois les informations notées, l'add-on peut être arrêté, voire désinstallé.

> ⚠️ **Important — stabilité de la local_key** : supprimer puis rajouter l'appareil **dans l'app Smart Life** génère une nouvelle `local_key` (l'appareil doit alors être repairé avec Wi-Fi, et toute intégration HA configurée avec l'ancienne clé cesse de fonctionner). Éviter de faire ça une fois l'appareil configuré dans Home Assistant. À l'inverse, supprimer l'app Smart Life elle-même du téléphone (ou se déconnecter/reconnecter au compte) n'a aucun effet sur la `local_key` — seule la suppression de l'appareil dans l'app en génère une nouvelle.

</details>

---

## 6. Lire les DPs de l'appareil

<details>
<summary>Voir plus</summary>

### Trouver la version du protocole

Avant de construire le script, déterminer la version du protocole utilisé par l'appareil (obligatoire — un mauvais numéro fait échouer la lecture silencieusement, sans message d'erreur). Dans le terminal de l'add-on :

```bash
docker exec -it homeassistant python3 -m tinytuya scan
```

Repérer la ligne correspondant à l'appareil et noter la version indiquée aisni que l'adresse IP locale de votre appareil :

```
Unknown v3.5 Device   Product ID = acqex5ltmos5fqed  [Valid Broadcast]:
    Address = 192.168.1.247   Device ID = ebdc2e75cfd943ea96e1om (len:22)  Local Key =   Version = 3.5
```

Ici, la version est **3.5** (visible au début de la ligne et dans le champ `Version =`). Prendre cette valeur en note, elle sera utilisée dans le script ci-dessous.
Ici, l'adresse IP locale est **192.168.1.247** (visible dans le champ `Address =`). Prendre cette valeur en note, elle sera utilisée dans le script ci-dessous.

### Lire les DPs

**Assurez-vous d'avoir en main ces 4 éléments, sinon relire les étapes précédentes:**
* Device ID
* Addresse IP Locale
* Local Key
* Version du protocole


Toujours dans le terminal de l'add-on :

```bash
docker exec -i homeassistant python3 <<'EOF'
import tinytuya

d = tinytuya.Device('DEVICE_ID', 'DEVICE_IP', "LOCAL_KEY")
d.set_version(VERSION)   # la version notée à l'étape du scan, ex: 3.5

data = d.status()
print(data)
EOF
```

Utiliser un étediteur de texte, par exemple Notepad++ copier/coller le script plus haut dans une nouvelle page de Notepad++ et remplacer les champs `DEVICE_ID`, `DEVICE_IP`, `LOCAL_KEY` et `VERSION` par les valeurs obtenues précédemment. Ensuite, copier/coller le script final contenant vos informations dans le terminal de l'add-on et appuyer sur Entrée.

> Le `<<'EOF'` (heredoc **quoté**) est important si la `local_key` contient des caractères spéciaux (`` ` ``, `$`, `&`, `;`...) — cela empêche le shell de les interpréter avant qu'ils atteignent Python.

Le résultat obtenu ressemble à `{'dps': {'1': True, '103': 1500, '106': True, '190': 3450, ...}}` — les identifiants de DP et leurs valeurs actuelles, mais sans nom ni description. Cette liste de DP correspond à tout les DP qui sont exposés localement sur votre réseau par votre appareil tuya. À noter que dans certain cas, certain DP peuvent être exposés par le firmware de l'appareil même s'il n'ont en fait pas de fonction utiles/attribué. Cela est attribuable au fait que plusieurs manufacturiers utilisent un "template" générique pour concevoir le firmware le type d'appareil sans toutefois tous les utiliser. Il est possible aussi que certain DP ne doivent pas être contrôlés par l'utilisateur (réglages d'usine).

Prendre en note les DPs dans un tableau (voir exemple à l'étape 8).

Les DP non-utilisés / non-attribués mais qui sont tout de même exposés, resterons statique (aucun changement de valeur) durant les prochaines étapes et pourront être retirés du tableau.

> ⚠️ **En l'absence de réponse** (dict vide `{}`) : vérifier d'abord que la version utilisée correspond bien à celle notée lors du scan — c'est la cause la plus fréquente. Si le scan n'a pas trouvé l'appareil du tout, vérifier aussi que l'adresse IP n'a pas changé (relancer le scan pour confirmer).

</details>

---

## 7. Monitoring live des DPs + déduction via l'app Smart Life

<details>
<summary>Voir plus</summary>

C'est ici que se déduit le rôle de chaque DP, sans jamais toucher au cloud Tuya et sans devoir utiliser de compte sur iot.tuya.com.

### Script d'écoute avec diff

```bash
docker exec -i homeassistant python3 <<'EOF'
import tinytuya
import time

d = tinytuya.Device('DEVICE_ID', 'DEVICE_IP', "LOCAL_KEY")
d.set_version(VERSION)  # la même version notée à l'étape 6
d.set_socketPersistent(True)

last = d.status().get('dps', {})
print(f"[{time.strftime('%H:%M:%S')}] État initial: {last}")

while True:
    data = d.receive()
    if data and 'dps' in data:
        changed = {k: v for k, v in data['dps'].items() if last.get(k) != v}
        if changed:
            print(f"[{time.strftime('%H:%M:%S')}] Changé: {changed}")
            last.update(data['dps'])
EOF
```

Commme pour l'étape précédente: Utiliser un éditeur de texte, par exemple Notepad++ copier/coller le script plus haut dans une nouvelle page de Notepad++ et remplacer les champs `DEVICE_ID`, `DEVICE_IP`, `LOCAL_KEY` et `VERSION` par les valeurs obtenues précédemment. Ensuite, copier/coller le script final contenant vos informations dans le terminal de l'add-on et appuyer sur Entrée.

Laisser ce terminal ouvert et actif pendant toute la session (`Ctrl+C` pour arrêter).

### Méthode de déduction

Le script actif, ouvrir l'app Smart Life et modifier **un seul réglage à la fois**, en notant ce qui apparaît dans le terminal.

**Type de donnée** — se lit directement dans la valeur brute reçue :
- `True` / `False` → **Boolean**
- Entier avec des valeurs intermédiaires possibles → **Integer**
- Entier qui ne prend jamais que quelques valeurs fixes même en testant les positions intermédiaires → **Enum déguisé en entier**
- Chaîne de caractères → **String** ou **Enum** texte

**Range (min/max)** — pousser le contrôle à ses deux extrêmes dans l'app (slider au minimum, puis au maximum) et noter la valeur brute à chaque bout. L'app respecte déjà les vraies limites du schema, donc ces deux essais donnent le min/max exacts sans risque de sortir des bornes réelles.

**Step / scale** — déplacer le slider d'un seul cran dans l'app et mesurer de combien la valeur brute a changé (= step). Comparer la valeur *affichée* dans l'app à la valeur *brute* reçue pour trouver le scale :
- L'app affiche "15.0 °C", brut = `150` → `scale = 1` (diviser par 10)
- L'app affiche "22 °C", brut = `22` → `scale = 0`

**Enum** — pour un contrôle à choix multiples (mode, vitesse...), sélectionner chaque option une par une dans l'app et noter la valeur brute correspondante pour construire la liste complète.

**Groupes de DPs consécutifs** (ex: paires `141/142`, `143/144`...) — souvent un programme horaire ou des réglages multi-zones. Changer un seul créneau/zone à la fois pour isoler quel DP correspond à quoi (heure, minute, jour, setpoint...).

> ⚠️ Éviter d'envoyer des valeurs hors plage directement via `d.set_value()` pour "tester les bornes" — certains firmwares n'ont aucune validation côté device et appliqueront la valeur brute telle quelle, ce qui peut dérégler l'appareil. Rester sur les limites imposées par l'app tant que le mapping n'est pas confirmé.

**Point de repère utile** : Tuya réserve généralement les DPs **1 à ~100** aux fonctions standard par catégorie de produit (voir [ressources](#9-ressources)) — si les DPs bas (1-20) suivent un pattern reconnaissable, cela confirme la catégorie de base de l'appareil. Les DPs **101+** sont presque toujours des extensions propriétaires propre à chaque fabricant, à déduire uniquement par cette méthode empirique.

**Exemple 1**

Dans cet exemple nous voulons trouver le DP qui correspond à la fonction "no load protection" d'une pompe tuya. On execute le script mentionné plus haut en prenant soin de bien renseigner le device id, adresse_ip, localkey et version de protocole. Par la suite dans l'app smart life nous basculerons la fonction "no load protection" de notre pompe entre activé et désactivé tout en vérifiant les informations dans le terminal. Nous voyons que tinytuya, via le script, détecte un changement sur le DP # 106 qui basule entre True et False. Alors le DP 106 est de type Bolean et correspond à la fonction "no load protection"

<img width="349" height="714" alt="image" src="https://github.com/user-attachments/assets/f1e1df88-5f70-4d39-9c7d-599c2ab8345c" /><img width="350" height="718" alt="image" src="https://github.com/user-attachments/assets/b074300e-2834-48fb-97ad-42874f80e336" />

<img width="690" height="498" alt="image" src="https://github.com/user-attachments/assets/306176a7-be1e-4cdf-a107-b1c634ad7fa9" />


**Exemple 2**

Dans cet exemple nous voulons trouver le DP qui correspond à la fonction "quick clean speed" d'une pompe tuya. On execute le script mentionné plus haut en prenant soin de bien renseigner le device id, adresse_ip, localkey et version de protocole. Par la suite dans l'app smart life nous changerons les valeurs possibles "quick clean speed" (valeure minimum et maximum) tout en vérifiant les informations dans le terminal. Nous voyons que tinytuya, via le script, détecte un changement sur le DP # 190 qui change de valeur entre 1000 et 3450. Alors le DP 190 est de type integer et correspond à la fonction "quick clean speed". Sa valeur minimale est 1000 et sa valeur maximale est 3450. Aussi, le minimum d'incrément possible via l'app smartlife et de 10 donc la valeur de "step / scale" est 10.

<img width="351" height="715" alt="image" src="https://github.com/user-attachments/assets/71e9cc87-60ff-407d-9b6d-8f9bc4180edf" /><img width="350" height="719" alt="image" src="https://github.com/user-attachments/assets/e705f88c-b04f-4fa4-99ac-7cf0851159dc" />

<img width="818" height="464" alt="image" src="https://github.com/user-attachments/assets/6de74de8-5427-47a6-b6f3-d58d1c7ce915" />

</details>

---

## 8. Tableau de suivi

<details>
<summary>Voir plus</summary>

Documenter chaque DP au fur et à mesure dans un tableau qui servira ensuite à configurer manuellement l'appareil tuya dans tuya-local ou localTuy (sans utilisation du cloud). Exemple abbrégé :

| DP  | Nom déduit        | Type    | Range       | Step / Scale  | Notes                          |
|-----|-------------------|---------|-------------|---------------|---------------------------------|
| 1   | switch            | Boolean | —           | —             | Marche/arrêt principal          |
| 106 | load protection   | Boolean | —           | —             | No load protection on/off       |
| 190 | quick clean speed | Integer | 1000-3450   | 10            | Vitesse RPM du quick clean      |

</details>

---

## 9. Ressources

<details>
<summary>Voir plus</summary>

- [tuya-local-key (récupération device_id / local_key sans cloud)](https://github.com/vineetchoudhary/tuya-local-key)
- [tinytuya](https://github.com/jasonacox/tinytuya)
- [Advanced SSH & Web Terminal (add-on)](https://github.com/hassio-addons/addon-ssh)
- [Tuya — Standard Instruction Set (par catégorie de produit)](https://developer.tuya.com/en/docs/iot/standarddescription?id=K9i5ql6waswzq)
- [Tuya — Device DPs (glossaire)](https://developer.tuya.com/en/docs/iot/device-function-point?id=Ka6y8bi672n1s)
- [tuya-local (configs communautaires par appareil, pour Home Assistant)](https://github.com/make-all/tuya-local)
- [xZetsubou/hass-localtuya (fork recommandé, compatible protocole 3.5)](https://github.com/xZetsubou/hass-localtuya)
- [rospogrigio/localtuya (dépôt original — non compatible 3.5)](https://github.com/rospogrigio/localtuya/)

</details>
