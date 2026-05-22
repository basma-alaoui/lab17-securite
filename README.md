Rétro-ingénierie de l'application OWASP Uncrackable Level 3
          
          📌 Présentation du laboratoire
Ce document décrit une analyse de sécurité menée sur l’application Android OWASP Uncrackable Level 3, dans un cadre strictement défensif et pédagogique. L’objectif est d’étudier les mécanismes de protection embarqués, de neutraliser les détections d’environnement rooté, puis d’extraire la clé secrète utilisée par la logique native pour valider le code utilisateur.
       
           🎯 Objectifs pédagogiques
À l’issue de ce travail, l’étudiant sera capable de :

Reconnaître les verrous de sécurité classiques sur Android

Contourner des tests de détection de root et de débogage

Modifier du code Smali manuellement

Reconstruire et signer un APK après modification

Analyser une bibliothèque native (.so) avec un désassembleur

Identifier des techniques d’obfuscation dans du code compilé

Extraire une chaîne protégée par XOR et la décoder

            ⚖️ Cadre légal et éthique
Ce travail est réalisé uniquement sur une application de test OWASP, dans un environnement personnel (émulateur ou périphérique dédié). Aucune utilisation malveillante ou diffusion de code modifié n’est autorisée.

           🧰 Outils mis en œuvre
Analyse statique Java : Jadx-GUI

Décompilation et reconstruction : apktool

Édition Smali : Visual Studio Code

Analyse native : Ghidra

Outils Android : adb, apksigner

            Scripting : Python 3

            1. Premier lancement et constat
L’application est installée sur un émulateur Android. Au démarrage, elle affiche immédiatement un message d’alerte :

“Rooting or tampering detected. This is unacceptable. The app is now going to exit.”

L’application se ferme ensuite automatiquement. Ce comportement indique la présence de contrôles d’intégrité et de détection d’environnement modifié.

            2. Exploration du bytecode Java avec Jadx
L’APK est chargé dans Jadx-GUI pour examiner le code source reconstitué. La classe MainActivity révèle plusieurs méthodes de vérification :

checkRoot1()

checkRoot2()

checkRoot3()

isDebuggable()

Si l’une de ces fonctions détecte une anomalie (root, débogage actif), la méthode showDialog() est invoquée pour bloquer l’application.

              3. Décompilation de l’APK avec apktool
Pour accéder au code Smali et aux ressources, on exécute :

bash
apktool d UnCrackable-Level3.apk
Le dossier généré contient le code modifiable.

            4. Neutralisation de la protection dans le Smali
Plutôt que de supprimer chaque test un par un, on cible la fonction showDialog() dans MainActivity.smali. On remplace son contenu par un simple retour, ce qui empêche l’affichage de l’alerte et la fermeture de l’application.

Modification appliquée : insertion de return-void en tout début de méthode.

              5. Reconstruction et signature de l’APK patché
Reconstruction
bash
apktool b uncrackable3 -o UnCrackable-Level3-patched.apk
Signature
bash
apksigner sign --ks debug.keystore UnCrackable-Level3-patched.apk
Installation
bash
adb install -r UnCrackable-Level3-patched.apk
Après cette opération, l’application démarre sans afficher l’alerte de sécurité.

              6. Analyse de la bibliothèque native libfoo.so
Le code Java montre que la validation finale du code utilisateur repose sur une fonction native. La bibliothèque libfoo.so est extraite de l’APK et analysée avec Ghidra.

Obfuscation constatée dans la fonction FUN_001012c0
Plusieurs techniques rendent la lecture difficile :

Control Flow Flattening (aplatissement du flux)

Blocs factices sans utilité réelle

Allocations mémoire superflues

Calculs parasites (type générateur congruentiel linéaire)

Structures conditionnelles artificielles

Protection de la clé secrète
La clé n’est pas présente en clair dans le binaire. Elle est reconstituée en mémoire par une série d’opérations XOR, puis comparée octet par octet à la saisie utilisateur.

Les données encodées extraites depuis le binaire sont :

text
1d 08 11 13 0f 17 49 15 0d 00 03 19 5a 1d 13 15 08 0e 5a 00 17 08 13 14
                
                7. Décodage de la clé secrète
La routine XOR exploite une clé répétée : pizzapizzapizzapizzapizzapizza.
Un script Python permet le déchiffrement :

python
data = bytes.fromhex("1d0811130f1749150d0003195a1d1315080e5a0017081314")
xor_key = b"pizzapizzapizzapizzapizzapizza"

result = bytes(d ^ k for d, k in zip(data, xor_key))
print(result.decode())
Le résultat obtenu est :

text
making owasp great again
Il s’agit de la chaîne attendue pour valider l’application.

            8. Remarques sur l’implémentation JNI
La méthode check.check_code() agit comme simple pont JNI. La logique métier réside entièrement dans la fonction native FUN_001012c0. Cela illustre l’intérêt de déporter du code critique vers une bibliothèque compilée.

            9. Intérêt des protections natives
Déplacer des mécanismes vers du C/C++ complique la rétro-ingénierie, car :

Le code compilé est plus difficile à analyser que le bytecode Java

L’obfuscation peut être plus agressive (flattening, calculs inutiles)

Les chaînes sensibles peuvent ne jamais apparaître en clair

             10. Pistes d’amélioration défensive
Pour renforcer encore l’application, un développeur pourrait implémenter :

De la cryptographie en boîte blanche (white-box)

Une vérification dynamique du certificat de signature

Une clé dépendante de l’environnement (ex : empreinte matérielle)

Des techniques anti-mémoire (effacement, chiffrement en RAM)

             11. Références utilisées
Ce laboratoire s’appuie sur les ressources suivantes :

Projet OWASP MASVS (niveau de résistance)

OWASP MASTG (tests et techniques)

            12. Captures d’écran (description)
Décompilation avec apktool – arborescence générée

Vue Jadx de MainActivity – mise en évidence des appels à checkRoot*

Application patchée – démarrage sans alerte

Script Python XOR – décodage de la clé

Validation finale – saisie de la clé dans l’application
