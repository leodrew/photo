# docker筆記
###### tags: `docker`
## docker install
```bash=
sudo apt-get update#先更新package
sudo apu-get install docker-ce=<Version>#安裝docker ce
```
## docker run & docker pull
- 使用 docker抓image通常有兩種指令一種是 docker pull 另外一種則是 docker run
```bash=
docker pull $image_name:version #這邊預設都是去 docker hub 抓取 
docker run $image_name:version #這邊預設都是去 docker hub 抓取 上網抓特定版本的image並且執行
docker pull <myregistry.local:5000/testing/test-image> #這邊是到特定的 registry 去抓取 image
```
:::warning
在上面講道說去特定的 registry 抓取 image 的時候通常會需要Registry credentials 這時候會需要使用 docker login
:::
## docker login
```bash=
docker login # docker 這樣打完之後會請你輸入帳號密碼，你在輸入你私人的 registry 的帳號密碼
docker login localhost:8080 #你也可以直接輸入你要抓取的網域
cat ~/my_password.txt | docker login --username foo --password-stdin #你可以透過非互動的方式來 login docker
```
![](https://i.imgur.com/x9z5PKF.png)
```bash=
docker ps #列出正在執行的container
```
![](https://i.imgur.com/dt6xN7z.png)
```bash=
docker images #列出所有本地端有抓過的images
```
## docker 常用指令
```bash=
docker run -d redis #-d意思是讓docker跑在背景程序，讓你可以繼續使用terminal
```
```bash=
docker stop <containerid>#停止特定的container
```
```bash=
docker ps -a#會列出所有的container包括未執行的
```
```bash=
docker run -p6000:6379 #設定forward port讓本地端的6000port對應到container的6379port
```
```bash=
docker logs <container id>/<container names>#查看container log紀錄
```
```bash=
docker run -d -p6001:6379 --name redis-older redis:4.0
#--name 可以自定義container name
```
```bash=
docker exec -it <container id>/<container name> /bin/bash
#可以進入到container裡面的終端機
```

## docker run & docker start差別
- `docker run`會產生一個新的container，假如你需要定義新的port mapping或者container name在使用這個
- `docker start`就是把舊的container重新啟動

## docker network
```bash=
docker network create `你想要取的網路名稱`
```
```bash=
docker network ls #會列出所有現在有的docker網路
```
![](https://i.imgur.com/ygiBgo2.png)
### mongodb示範
```bash=
docker run -p 27017:27017 -d  \
-e MONGO_INITDB_ROOT_USERNAME=admin \
-e MONGO_INITDB_ROOT_PASSWORD=password \
--name mongodb --network mongo-network mongo
# -e 設定環境變數 --network把container指定到一個網路裡面
```
### mongo-express示範
```bash=
docker run -d \
-p 8081:8081 \
-e ME_CONFIG_MONGODB_ADMINUSERNAME=admin \ #這要跟你當初設資料庫的root_username一樣
-e ME_CONFIG_MONGODB_ADMINPASSWORD=password\
--net mongo-network \
--name mongo-express \
-e ME_CONFIG_MONGODB_SERVER=mongodb \ #這個是當初設定mongo的container name       
mongo-express
```

## docker run command && docker compose差異
- 假如你今天有多個服務要互相溝通的話就可以用docker compose
- 可以簡化每一次冗長的docker run指令
- 在docker-compose裡面不用特定設定network它會自動把你寫在同一個yaml黨裡面的服務自動歸類在同一個網路裡面
```yaml=
version: '3' #docker-compose版本
services:
  mongodb: #這個等於上面docker run --name的用法
    image: mongo #這個是說你要用哪個image
    ports:
      - 27017:27017
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=password
```

```bash=
docker-compose -f mongo.yaml up -d
# -f 如果你要使用特定檔案啟動docker-compose 要使用-f
# -d 依樣是讓程序跑在背景
```

## dockerfile
- dockerfile的用意是讓你打造專屬於你開發環境的image

```dockerfile=
FROM node:<version> #假如你需要仔特定版本的image
ENV MONGO_INITDB_ROOT_USERNAME=admin \
    MONGO_INITDB_ROOT_PASSWORD=password #設定環境變數但建議是在docker-compose上面設定因為假如你要變動的話可以直接在docker-compose上面變動要不然你要刪掉image再重新build一次
RUN mkdir -p /home/app #你可以執行任何linux指令透過run
COPY . /home/app #你可以把本地端的檔案複製到container
CMD ["node", "server.js"] #當你使用這個image的時候他預設就會執行`node sever.js`這個指令
```
```bash=
docker build -t my-app:1.0 .
# -t 意思用來幫你image取名子
```

## docker repository
- 這邊主要是要把你寫好的image放到一個平台上面，類似git repository
- 這邊是使用aws ecr示範
- 這邊要注意假設你要把image push上去 你要先docker login 
- 第二點是假設你要使用ci的方式你也要給他權限

### image naming in docker refistries
- 通常這種平台上面有image註冊名子的格式如下面示範
![](https://i.imgur.com/XElLaNM.png)
- 通常我們從docker hub官方載image只要打`docker pull mongo:4.2`的原因是因為docker它內建有幫我們寫好config所以可以這樣用
- 但假如今天是要從私人平台上面pull image的話就要打上全名如下面這樣
![](https://i.imgur.com/CT5nfCe.png)
- 所以當你今天要push到aws你要按照他的方式要先幫妳的image重新tag
![](https://i.imgur.com/sdXkfGB.png)
```bash=
docker tag my-app:1.0 664574038682.dkr.ecr.eu-central-1.amazonaws.com/my-app:1.0
#前面這一段`docker tag my-app:1.0`意思是將你本端的my-app重新rename
```
- docker tag完之後，最後在push上aws ecr
```bash=
docker push 664574038682.dkr.ecr.eu-central-1.amazonaws.com/my-app:1.0
```
```yaml=
version: '3' #docker-compose版本
services:
  my-app: 
    image: 664574038682.dkr.ecr.eu-central-1.amazonaws.com/my-app:1.0 #如果你假設要從私人的平台載image要這樣打
  mongodb: #這個等於上面docker run --name的用法
    image: mongo #這個是說你要用哪個image
    ports:
      - 27017:27017
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=password
```

## docker volumes:
- 這邊主要是不要讓每次重啟db服務的時候資料就不見了，因此docker volumes就是來解決這種問題


```yaml=
version: '3' #docker-compose版本
services:
  my-app: 
    image: 664574038682.dkr.ecr.eu-central-1.amazonaws.com/my-app:1.0 #如果你假設要從私人的平台載image要這樣打
  mongodb: #這個等於上面docker run --name的用法
    image: mongo #這個是說你要用哪個image
    ports:
      - 27017:27017
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=password
    volumes:
      - mongo-data:/data/db #這邊是mongo db預設存資料的地方，然而每個資料庫不一樣在使用上要先查清楚
volumes:
  mongo-data:
```
- 而這些volumes會存在本地端的哪裡會根據系統不一樣有不同的路徑，詳細如下。
![](https://i.imgur.com/PhCVxfa.png)
### docker volumes type
volumes其實有三種型式
- 第一種是定義好完整的路徑
`/home/mount/data:/var/lib/mysql/data`
- 第二種是匿名是路徑 這種型態docker自己會隨機生成名子存到本地端
`/var/lib/mysql/data`
- 第三種是匿名路徑升級本，可以自己定義name讓後續比較好找
`name:/var/lib/mysql/data`
![](https://i.imgur.com/0u1O0z1.jpg)

## docker entrypoint & docker cmd
有時候在dockerfile裡面會看到 entrypoint cmd這兩個區別主要是當你執行`docker run -it ubuntu <command>`他是根據entrypoint再加上 command去執行那段指令

假設今天有img的entrypoint是["/bin/cat"] CMD是[123.txt]
- 第一種情況是`docker run img`當什麼都不打的時候他預設是印出123.txt的內容
- 那`docker run img /etc/passwd`這個指令在container完整的指令是長下面這樣`/bin/cat /etc/passwd.`
