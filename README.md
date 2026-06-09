# Déploiement de clients légers NixOS via miroir Gitea

Documentation du workflow de déploiement de configurations NixOS sur des clients légers, à partir d'un dépôt GitHub mis en miroir sur un serveur Gitea auto-hébergé (NAS Synology).

## Architecture

```
GitHub (source de vérité)
   │  pull mirror (toutes les 10 min)
   ▼
Gitea sur NAS Synology (miroir lecture seule)
   │  git clone / git pull
   ▼
Clients légers NixOS (déploiement via flake)
```

- **GitHub** héberge la configuration NixOS (source de vérité, en flakes).
- **Gitea** (conteneur Docker sur le NAS) en maintient un miroir en lecture seule, synchronisé automatiquement.
- Les **clients légers** clonent depuis le Gitea local (pas depuis GitHub) pour déployer leur configuration.

## Prérequis

- Serveur Gitea déployé et fonctionnel sur le NAS, accessible à `https://git.local.*****.org:3000`.
- Compte administrateur Gitea créé.
- Configuration NixOS en flakes hébergée sur GitHub.
- Clé USB d'installation NixOS.

## Structure du dépôt de configuration

```
flake.nix
hosts/
  client-leger-01/
    hardware-configuration.nix   # réel, committé depuis le poste de gestion
  client-leger-02/
    hardware-configuration.nix
modules/
  common.nix                     # configuration partagée (réseau, users, paquets…)
```

Chaque hôte possède son propre `hardware-configuration.nix`. Les éléments communs sont factorisés dans `modules/`.

---

## 1. Mise en place du miroir dans Gitea

À effectuer une seule fois, depuis l'interface web de Gitea (connecté en administrateur).

1. Cliquer sur **`+`** (en haut à droite) → **New Migration**.
2. Choisir **GitHub** comme source.
3. Renseigner les champs :
   - **Clone Address** : URL du dépôt GitHub, ex. `https://github.com/<utilisateur>/nixos-config`
   - **Access Token** : token GitHub si le dépôt est privé (GitHub → `Settings → Developer settings → Personal access tokens`, scope `repo`). Inutile pour un dépôt public.
4. Cocher impérativement **`This repository will be a mirror`**.
5. Nommer le dépôt côté Gitea (ex. `nixos-config`).
6. Valider.

Gitea clone immédiatement le dépôt, puis le resynchronise automatiquement selon l'intervalle configuré (`DEFAULT_INTERVAL`, par défaut 10 min).

Pour forcer une synchronisation immédiate : **dépôt → `Settings → Mirror Settings → Synchronize Now`**.

---

## 2. Premier allumage d'un client léger (installation complète)

À effectuer sur une machine vierge.

### 2.1 Booter sur le live ISO NixOS

Booter le client léger sur la clé USB d'installation NixOS. On arrive sur un environnement live avec un shell root.

### 2.2 Partitionner et formater le disque

Exemple en UEFI (adapter le disque cible, vérifié avec `lsblk`) :

```bash
# Repérer le disque
lsblk

# Partitionnement GPT/UEFI sur /dev/sda
sudo parted /dev/sda -- mklabel gpt
sudo parted /dev/sda -- mkpart ESP fat32 1MiB 512MiB
sudo parted /dev/sda -- set 1 esp on
sudo parted /dev/sda -- mkpart primary 512MiB 100%

# Formater
sudo mkfs.fat -F 32 -n boot /dev/sda1
sudo mkfs.ext4 -L nixos /dev/sda2
```

### 2.3 Monter les partitions

```bash
sudo mount /dev/disk/by-label/nixos /mnt
sudo mkdir -p /mnt/boot
sudo mount /dev/disk/by-label/boot /mnt/boot
```

### 2.4 Récupérer la configuration depuis le NAS

```bash
nix-shell -p git
git clone https://git.local.*******.org:3000/admin/nixos-config.git /mnt/etc/nixos-config
cd /mnt/etc/nixos-config
```

### 2.5 Générer le hardware-configuration réel

Les partitions étant montées sous `/mnt`, la génération détecte la vraie disposition matérielle :

```bash
sudo nixos-generate-config --root /mnt --show-hardware-config \
  > hosts/<nom_de_lhote>/hardware-configuration.nix
```

Cette étape remplace le placeholder par le matériel réel de la machine.

### 2.6 Installer avec la configuration flake

```bash
sudo nixos-install --flake /mnt/etc/nixos-config#<nom_de_lhote>
```

`nixos-install` build le système, l'installe sur `/mnt`, puis demande de définir le mot de passe root.

### 2.7 Redémarrer

```bash
sudo reboot
```

Retirer la clé USB. La machine boote sur NixOS avec la configuration appliquée.

---

## 3. Mises à jour d'une machine déjà déployée

Sur un client déjà installé, pour appliquer les modifications synchronisées depuis GitHub :

```bash
cd /etc/nixos-config
git pull
sudo nixos-rebuild switch --flake .#<nom_de_lhote>
```

---

## Gestion du hardware-configuration

Le `hardware-configuration.nix` est **propre à chaque machine** et ne doit jamais être un placeholder partagé en production.

Comme le dépôt Gitea est un **miroir en lecture seule**, le hardware régénéré sur un client ne peut pas y être poussé. Le fichier réel doit donc être committé sur **GitHub depuis le poste de gestion**, pas depuis le client.

Deux organisations possibles :

**Option A — un fichier hardware par hôte (parc hétérogène)**
Après provisionnement d'une machine, récupérer son `hardware-configuration.nix` généré et le commiter sur GitHub depuis le poste de gestion. Le miroir se synchronise, la machine n'a plus qu'à faire `git pull`. Chaque hôte a son fichier réel versionné.

**Option B — modèle matériel unique**
Si tous les clients légers sont matériellement identiques, générer le hardware-config une seule fois et le commiter comme référence partagée. Toutes les machines utilisent la même définition, et l'étape 2.5 disparaît du premier allumage (l'installation se réduit à clone + `nixos-install`).

---

## Référence rapide des commandes

| Action | Commande |
| --- | --- |
| Forcer la synchro du miroir | Gitea → dépôt → `Settings → Mirror Settings → Synchronize Now` |
| Cloner la config (install) | `git clone https://git.local.******.org:3000/admin/nixos-config.git /mnt/etc/nixos-config` |
| Générer le hardware réel | `sudo nixos-generate-config --root /mnt --show-hardware-config > hosts/<hôte>/hardware-configuration.nix` |
| Installer | `sudo nixos-install --flake /mnt/etc/nixos-config#<hôte>` |
| Mettre à jour | `cd /etc/nixos-config && git pull && sudo nixos-rebuild switch --flake .#<hôte>` |
