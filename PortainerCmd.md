## Portainer Kurulumu? 

Volume Oluþturma:
````
docker volume create portainer_data
````
 Portainer Ýçin Gerekli Kod:
````
docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
````
* Sonrasýnda tarayýcýda **localhost:9000** adresinde portainer arayüzü bizi karþýlýyor.