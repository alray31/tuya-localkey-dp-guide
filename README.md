# Trouver ses LocalKey, DeviceID et DPs de ses appareils Tuya wifi — depuis Home Assistant OS - sans devoir utiliser de compte dev sur iot.tuya.com

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

Le `device_id` et la `local_key` récupérés ici servent à ajouter manuellement un appareil WiFi Tuya dans Home Assistant via une intégration custom HACS en local — **sans passer par le cloud Tuya à l'usage**, et surtout **sans avoir besoin de créer de compte ni de projet développeur sur `iot.tuya.com`**. Deux intégrations HACS courantes utilisent ces informations :

- **[tuya-local](https://github.com/make-all/tuya-local)** — configs par appareil basées sur des profils YAML communautaires.
- **[localtuya](https://github.com/rospogrigio/localtuya/)** — configuration manuelle DP par DP, entité par entité.

Le mapping des DPs (nom, type, range, step) construit dans ce guide sert précisément à configurer correctement ces entités dans l'intégration choisie, une fois le `device_id`/`local_key` en main.

> ⚠️ **Si vous utilisez localtuya** : préférez le fork **[xZetsubou/hass-localtuya](https://github.com/xZetsubou/hass-localtuya)** au dépôt original `rospogrigio/localtuya` listé plus haut. Le fork est mieux maintenu et compatible avec le protocole **3.5**, alors que le dépôt original ne l'est pas — un appareil récent utilisant le protocole TuYa 3.5 restera invisible ou impossible à configurer avec la version originale de localtuya.

---

## 2. Prérequis

- Home Assistant OS (ou Supervised), avec accès au panneau **Paramètres → Add-ons**.
- L'appareil Tuya déjà appairé et fonctionnel dans l'app **Smart Life** (ou Tuya Smart) sur un téléphone.
- Un téléphone à portée, avec l'app smartlife installée, pour scanner un QR code et pour manipuler l'appareil tuya pendant le monitoring.

Aucun compte développeur sur `iot.tuya.com` n'est nécessaire pour ce guide.

> ⚠️ Ce guide donne un accès complet à l'hôte Docker de l'installation HA (protection mode désactivé sur l'add-on). C'ela nécessaire pour les commandes `docker exec`/`docker run` ci-dessous, mais cela sort du cadre "sécurisé par défaut" de HAOS — restez prudent avec les commandes exécutées, et désactivez/retirez l'add-on une fois le travail terminé, si désiré, pour revenir à un système plus verrouillé.

---

## 3. Installer l'add-on Advanced SSH & Web Terminal

L'add-on officiel "Terminal & SSH" ne donne pas accès à Docker. Il faut l'add-on communautaire **Advanced SSH & Web Terminal**, qui donne un véritable shell sur l'hôte (avec `docker`).

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

Un shell avec accès complet à Docker est maintenant disponible, directement depuis le navigateur.

---

## 4. Vérifier / installer tinytuya dans le conteneur Home Assistant

Dans le terminal de l'add-on :

```bash
docker exec -it homeassistant pip show tinytuya
```

S'il est absent :

```bash
docker exec -it homeassistant pip install tinytuya
```

> ⚠️ Une installation manuelle dans le conteneur ne survit pas à une recréation du conteneur (mise à jour de l'image HA). Pour une utilisation ponctuelle de reverse engineering, ce n'est pas un problème — il suffit de le réinstaller au besoin.

---

## 5. Récupérer device_id et local_key avec tuya-local-key

[`tuya-local-key`](https://github.com/vineetchoudhary/tuya-local-key) récupère la liste des appareils (ID, local key, IP, catégorie...) via une connexion QR code au compte Smart Life — **sans créer de projet développeur Tuya**. Il existe en version add-on Home Assistant.

### Installation (en tant qu'add-on)

1. **Paramètres → Add-ons → Boutique d'add-ons**.
2. Menu **⋮** (en haut à droite) → **Dépôts** → ajouter :
   ```
   https://github.com/vineetchoudhary/tuya-local-key
   ```
3. Rechercher **"Tuya Local Key"** dans la liste et cliquer sur **Installer**.
4. Une fois installé, le démarrer, puis l'ouvrir directement depuis la barre latérale de Home Assistant (l'add-on utilise l'ingress de Home Assistant — pas besoin de connaître une IP ou un numéro de port).

### Utilisation

1. Dans l'app Smart Life sur votre téléphone: **Moi → Paramètres → Compte et sécurité → Code utilisateur**. Noter ce code et l'entrer dans l'interface de l'add-on.

<img width="240" height="500" alt="image" src="https://github.com/user-attachments/assets/50c43fa8-71ce-469b-8191-6b285b99c90d" />

2. Un QR code s'affiche dans l'interface de l'add-on. Le scanner avec l'app Smart Life :

<img width="245" height="500" alt="image" src="https://github.com/user-attachments/assets/37720f98-b78f-4295-a389-7375bcd243cf" />

3. Une demande de connexion apparaît dans l'app — appuyer sur **"Confirmer la connexion"**.
4. La liste des appareils apparaît dans l'add-on : `device_id`, `local_key`, `uuid`, `catégorie`, IP locale, statut en ligne.
5. Prendre en note les  `device_id`, `local_key`. Ne pas confondre `device_id` (appelé simplement "id" dans cet addon avec le `procduct id`

<img width="1591" height="357" alt="image" src="https://github.com/user-attachments/assets/f6e04bdf-4db6-4241-bb04-04b522a18882" />


> ⚠️ **Note** : Une fois les informations notées, l'add-on peut être arrêté, voire désinstallé.

> ⚠️ **Important — stabilité de la local_key** : supprimer puis rajouter l'appareil **dans l'app Smart Life** génère une nouvelle `local_key` (l'appareil doit alors être repairé avec Wi-Fi, et toute intégration HA configurée avec l'ancienne clé cesse de fonctionner). Éviter de faire ça une fois l'appareil configuré dans Home Assistant. À l'inverse, supprimer l'app Smart Life elle-même du téléphone (ou se déconnecter/reconnecter au compte) n'a aucun effet sur la `local_key` — seule la suppression de l'appareil dans l'app en génère une nouvelle.

---

## 6. Lire les DPs de l'appareil

Toujours dans le terminal de l'add-on :

```bash
docker exec -i homeassistant python3 <<'EOF'
import tinytuya

d = tinytuya.Device('DEVICE_ID', 'DEVICE_IP', 'LOCAL_KEY')
d.set_version(3.3)   # essayer 3.3, puis 3.4, puis 3.5 si aucune réponse

data = d.status()
print(data)
EOF
```

Remplacer `DEVICE_ID`, `DEVICE_IP` et `LOCAL_KEY` par les valeurs obtenues à l'étape précédente.

> Le `<<'EOF'` (heredoc **quoté**) est important si la `local_key` contient des caractères spéciaux (`` ` ``, `$`, `&`, `;`...) — cela empêche le shell de les interpréter avant qu'ils atteignent Python.

Le résultat obtenu ressemble à `{'dps': {'1': True, '103': 1500, ...}}` — les identifiants de DP et leurs valeurs actuelles, mais sans nom ni description (cette métadonnée n'existe que dans le cloud Tuya, inaccessible ici).

**En l'absence de réponse** : la version de protocole (3.1/3.3/3.4/3.5) est peut-être différente. Un scan réseau aide à la détecter :

```bash
docker exec -it homeassistant python3 -m tinytuya scan
```

---

## 7. Monitoring live des DPs + déduction via l'app Smart Life

C'est ici que se déduit le rôle de chaque DP, sans jamais toucher au cloud Tuya.

### Script d'écoute avec diff

```bash
docker exec -i homeassistant python3 <<'EOF'
import tinytuya
import time

d = tinytuya.Device('DEVICE_ID', 'DEVICE_IP', 'LOCAL_KEY')
d.set_version(3.3)  # à adapter selon l'appareil
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

**Point de repère utile** : Tuya réserve généralement les DPs **1 à ~100** aux fonctions standard par catégorie de produit (voir [ressources](#9-ressources)) — si les DPs bas (1-20) suivent un pattern reconnaissable, cela confirme la catégorie de base de l'appareil. Les DPs **101+** sont presque toujours des extensions propriétaires du fabricant, à déduire uniquement par cette méthode empirique.

Exemple 1:
Dans cet exemple nous voulons trouver le DP qui correspond à la fonction "no load protection" d'une pompe tuya. On execute le script mentionné plus haut en prenant soin de bien renseigner le device id, adresse_ip, localkey et version de protocole. Par la suite dans l'app smart life nous basculerons la fonction "no load protection" de notre pompe entre activé et désactivé tout en vérifiant les informations dans le terminal. Nous voyons que tinytuya, via le script, détecte un changement sur le DP # 106 qui basule entre True et False. Alors le DP 106 est de type Bolean et correspond à la fonction "no load protection"

<img width="349" height="714" alt="image" src="https://github.com/user-attachments/assets/f1e1df88-5f70-4d39-9c7d-599c2ab8345c" /><img width="350" height="718" alt="image" src="https://github.com/user-attachments/assets/b074300e-2834-48fb-97ad-42874f80e336" />


<img width="690" height="498" alt="image" src="https://github.com/user-attachments/assets/306176a7-be1e-4cdf-a107-b1c634ad7fa9" />

Exemple 2:
Dans cet exemple nous voulons trouver le DP qui correspond à la fonction "quick clean speed" d'une pompe tuya. On execute le script mentionné plus haut en prenant soin de bien renseigner le device id, adresse_ip, localkey et version de protocole. Par la suite dans l'app smart life nous changerons les valeurs possibles "quick clean speed" (valeure minimum et maximum) tout en vérifiant les informations dans le terminal. Nous voyons que tinytuya, via le script, détecte un changement sur le DP # 190 qui change de valeur entre 1000 et 3450. Alors le DP 190 est de type integer et correspond à la fonction "quick clean speed". Sa valeur minimale est 1000 et sa valeur maximale est 3450. Aussi, le minimum d'incrément possible via l'app smartlife et de 10 donc la valeur de "step / scale" est 10.

<img width="351" height="715" alt="image" src="https://github.com/user-attachments/assets/71e9cc87-60ff-407d-9b6d-8f9bc4180edf" /><img width="350" height="719" alt="image" src="https://github.com/user-attachments/assets/e705f88c-b04f-4fa4-99ac-7cf0851159dc" />



<img width="818" height="464" alt="image" src="https://github.com/user-attachments/assets/6de74de8-5427-47a6-b6f3-d58d1c7ce915" />


---

## 8. Tableau de suivi

Documenter chaque DP au fur et à mesure dans un tableau qui servira ensuite à configurer l'appareil dans tuya-local ou localTuya. Exemple abbrégé :

| DP  | Nom déduit        | Type    | Range       | Step / Scale  | Notes                          |
|-----|-------------------|---------|-------------|---------------|---------------------------------|
| 1   | switch            | Boolean | —           | —             | Marche/arrêt principal          |
| 106 | load protection   | Boolean | —           | —             | No load protection on/off       |
| 190 | quick clean speed | Integer | 1000-3450   | —             | À confirmer via test des modes  |

---

## 9. Ressources

- [tuya-local-key (récupération device_id / local_key sans cloud)](https://github.com/vineetchoudhary/tuya-local-key)
- [tinytuya](https://github.com/jasonacox/tinytuya)
- [Advanced SSH & Web Terminal (add-on)](https://github.com/hassio-addons/addon-ssh)
- [Tuya — Standard Instruction Set (par catégorie de produit)](https://developer.tuya.com/en/docs/iot/standarddescription?id=K9i5ql6waswzq)
- [Tuya — Device DPs (glossaire)](https://developer.tuya.com/en/docs/iot/device-function-point?id=Ka6y8bi672n1s)
- [tuya-local (configs communautaires par appareil, pour Home Assistant)](https://github.com/make-all/tuya-local)
- [xZetsubou/hass-localtuya (fork recommandé, compatible protocole 3.5)](https://github.com/xZetsubou/hass-localtuya)
- [rospogrigio/localtuya (dépôt original — non compatible 3.5)](https://github.com/rospogrigio/localtuya/)
