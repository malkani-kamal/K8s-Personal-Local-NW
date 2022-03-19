Step 1
------
kubectl apply -f ./storage/pvc.yaml 

Verify
------
kubectl get pvc
kubectl get pv


Create test containers to check the PVC
---------------------------------------
kubectl apply -f ./storage/tests
It will start 2 deployments containers where we will be creating folders i.e. host, chaincode, scripts.



kubectl get pods

To get into the containers
--------------------------
kubectl exec -it <containername> bash

Create two folders inside containers

cd /host
mkdir files
cd files
mkdir chaincode
mkdir scripts
copy bin from fabric samples to host/files/bin

Bin Folder contains
-------------------
root@example2-6d5bc76fd4-6j4vk:/host/files/bin# ls
configtxgen  configtxlator  cryptogen  discover  fabric-ca-client  fabric-ca-server  idemixgen	orderer  osnadmin  peer
root@example2-6d5bc76fd4-6j4vk:/host/files/bin# 


Chaincode Folder contains
-------------------------
root@example2-6d5bc76fd4-6j4vk:/host/files/chaincode# ls
resource_types	resources
root@example2-6d5bc76fd4-6j4vk:/host/files/chaincode# 


Files Folder contains
---------------------
root@example2-6d5bc76fd4-6j4vk:/host/files# ls
bin  chaincode	channels  config.yaml  configtx.yaml  crypto-config  orderer  scripts




Exit the container
-------------------

kubectl cp ./scripts example1-8588894689-jlxp4:/host/files

kubectl cp ./chaincode example1-8588894689-jlxp4:/host/files

kubectl cp ./network/production/configtx.yaml example1-8588894689-jlxp4:/host/files
kubectl cp ./network/production/config.yaml example1-8588894689-jlxp4:/host/files
kubectl cp ./../fabric-samples/bin example1-8588894689-jlxp4:/host/files

kubectl apply -f network/minikube/cas

kubectl exec -it <containername> bash

rm -rf orderer channels
mkdir -p orderer channels



bin/configtxgen -profile OrdererGenesis -channelID syschannel -outputBlock ./orderer/genesis.block
bin/configtxgen -profile MainChannel -outputCreateChannelTx ./channels/mainchannel.tx -channelID mainchannel
bin/configtxgen -profile MainChannel -outputAnchorPeersUpdate ./channels/ibm-anchors.tx -channelID mainchannel -asOrg ibm
bin/configtxgen -profile MainChannel -outputAnchorPeersUpdate ./channels/oracle-anchors.tx -channelID mainchannel -asOrg oracle


cd orderer
ls

Verify if the genesis block is created or no

exit the container and reach out side the container

Start orderer
-------------
kubectl apply -f network/minikube/orderers


Start couchdb
-------------
kubectl apply -f network/minikube/orgs/ibm/couchdb 
kubectl apply -f network/minikube/orgs/oracle/couchdb

Start Peers
-----------
kubectl apply -f network/minikube/orgs/ibm/
kubectl apply -f network/minikube/orgs/oracle/

Start CLI
---------
kubectl apply -f network/minikube/orgs/ibm/cli
kubectl apply -f network/minikube/orgs/oracle/cli

If you want to delete any container just `kubectl delete -f network/minikube/orgs/ibm/cli`

Create channel

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer channel create -c mainchannel -f ./channels/mainchannel.tx -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem'


Move mainchannel.block file to channels folder as it is currently in one container and outside the channel folder.

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'cp mainchannel.block ./channels/'


Join  (Run 1 by 1)
 

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer channel join -b channels/mainchannel.block'

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer1-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer channel join -b channels/mainchannel.block'

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-oracle-deployment | sed "s/^.\{4\}//") -- bash -c 'peer channel join -b channels/mainchannel.block'

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer1-oracle-deployment | sed "s/^.\{4\}//") -- bash -c 'peer channel join -b channels/mainchannel.block'


Update anchors

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer channel update -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem -c mainchannel -f channels/ibm-anchors.tx'

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-oracle-deployment | sed "s/^.\{4\}//") -- bash -c 'peer channel update -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem -c mainchannel -f channels/oracle-anchors.tx'



Create chaincode dependency package

Now we are going to install the chaincode - NOTE: Make sure you go mod vendor in each chaincode folder... might need to remove the go.sum depending
```bash


kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode package resource_types.tar.gz --path /opt/gopath/src/resource_types --lang golang --label resource_types_1'

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer1-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode package resource_types.tar.gz --path /opt/gopath/src/resource_types --lang golang --label resource_types_1'

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-oracle-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode package resource_types.tar.gz --path /opt/gopath/src/resource_types --lang golang --label resource_types_1'

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer1-oracle-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode package resource_types.tar.gz --path /opt/gopath/src/resource_types --lang golang --label resource_types_1'




Install chaincode


kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode install resource_types.tar.gz &> pkg.txt'

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer1-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode install resource_types.tar.gz'

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-oracle-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode install resource_types.tar.gz &> pkg.txt'

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer1-oracle-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode install resource_types.tar.gz'



Approve my chaincode

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode approveformyorg -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem --collections-config /opt/gopath/src/resource_types/collections-config.json --channelID mainchannel --name resource_types --version 1.0 --sequence 1 --package-id $(tail -n 1 pkg.txt | awk '\''NF>1{print $NF}'\'')'


kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-oracle-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode approveformyorg -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem --collections-config /opt/gopath/src/resource_types/collections-config.json --channelID mainchannel --name resource_types --version 1.0 --sequence 1 --package-id $(tail -n 1 pkg.txt | awk '\''NF>1{print $NF}'\'')'




Commit chaincode

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode commit -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem --collections-config /opt/gopath/src/resource_types/collections-config.json --channelID mainchannel --name resource_types --version 1.0 --sequence 1'






Lets go ahead and test this chaincode



kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer chaincode invoke -C mainchannel -n resource_types -c '\''{"Args":["Create", "1","Parts"]}'\'' -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem'


sleep 5

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer chaincode query -C mainchannel -n resource_types -c '\''{"Args":["Index"]}'\'' -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem'





























Install Second chaincode



Lets try the other chaincode


kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode package resources.tar.gz --path /opt/gopath/src/resources --lang golang --label resources_1'

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer1-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode package resources.tar.gz --path /opt/gopath/src/resources --lang golang --label resources_1'


kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-oracle-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode package resources.tar.gz --path /opt/gopath/src/resources --lang golang --label resources_1'

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer1-oracle-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode package resources.tar.gz --path /opt/gopath/src/resources --lang golang --label resources_1'



Install chaincode

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode install resources.tar.gz &> pkg.txt'


kubectl exec -it $(kubectl get pods -o=name | grep cli-peer1-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode install resources.tar.gz'


kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-oracle-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode install resources.tar.gz &> pkg.txt'


kubectl exec -it $(kubectl get pods -o=name | grep cli-peer1-oracle-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode install resources.tar.gz'

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode approveformyorg -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem --collections-config /opt/gopath/src/resources/collections-config.json --channelID mainchannel --name resources --version 1.0 --sequence 1 --package-id $(tail -n 1 pkg.txt | awk '\''NF>1{print $NF}'\'')'


kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-oracle-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode approveformyorg -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem --collections-config /opt/gopath/src/resources/collections-config.json --channelID mainchannel --name resources --version 1.0 --sequence 1 --package-id $(tail -n 1 pkg.txt | awk '\''NF>1{print $NF}'\'')'

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer lifecycle chaincode commit -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem --collections-config /opt/gopath/src/resources/collections-config.json --channelID mainchannel --name resources --version 1.0 --sequence 1'

sleep 5

kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer chaincode invoke -C mainchannel -n resources -c '\''{"Args":["Create","2", "CPUs","1"]}'\'' -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem'


kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer chaincode invoke -C mainchannel -n resources -c '\''{"Args":["Create","1", "Database Servers", "1"]}'\'' -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem'



kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-oracle-deployment | sed "s/^.\{4\}//") -- bash -c 'peer chaincode invoke -C mainchannel -n resources -c '\''{"Args":["Create","3", "Mainframes","1"]}'\'' -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem'
sleep 5


kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer chaincode query -C mainchannel -n resources -c '\''{"Args":["Index"]}'\'' -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem'
kubectl exec -it $(kubectl get pods -o=name | grep cli-peer1-ibm-deployment | sed "s/^.\{4\}//") -- bash -c 'peer chaincode query -C mainchannel -n resources -c '\''{"Args":["Index"]}'\'' -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem'
kubectl exec -it $(kubectl get pods -o=name | grep cli-peer0-oracle-deployment | sed "s/^.\{4\}//") -- bash -c 'peer chaincode query -C mainchannel -n resources -c '\''{"Args":["Index"]}'\'' -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem'
kubectl exec -it $(kubectl get pods -o=name | grep cli-peer1-oracle-deployment | sed "s/^.\{4\}//") -- bash -c 'peer chaincode query -C mainchannel -n resources -c '\''{"Args":["Index"]}'\'' -o orderer0-service:7050 --tls --cafile=/etc/hyperledger/orderers/msp/tlscacerts/orderers-ca-service-7054.pem'



kubectl apply -f network/production/backend/

check the url

minikube service api-service-nodeport  --url

curl http://192.168.59.100:31704/resource_types

output should be
[{"id":"1","name":"Parts","active":true,"privateName":""}]




kubectl apply -f network/production/frontend
