# Rootless Docker 
Create docker group if not exist
```
sudo groupadd docker 
```
Add user to docker group : 
```
sudo usermod -aG docker ${USER} 
```
Change docker.sock to new permission
```
sudo chmod 666 /var/run/docker.sock 
``` 
Finally, restart docker daemon service
```
sudo systemctl restart docker
```
# NFS Docker Volume
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=[ip-address],rw \
  --opt device=:[path-to-directory] \
  [volume-name]
