## Portainer Kurulumu? 

Volume Olu�turma:
````
docker volume create portainer_data
````
 Portainer ��in Gerekli Kod:
````
docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
````
* Sonras�nda taray�c�da **localhost:9000** adresinde portainer aray�z� bizi kar��l�yor.