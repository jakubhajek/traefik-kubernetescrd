Create user file

htpasswd -c userfile admin

kubectl create secret generic admin-authsecret --from-file=userfile
