# Deploy Palworld server

## Prepare Persistent Volume

`kubectl apply -f palworld-server/pv.yaml`

## Install server using helm

`helm install palworld ./palworld-server --set secret.rconPassword=yourRconPassword --set configmap.SERVER_PASSWORD=yourServerPassword --set env.ADMIN_PASSWORD=yourAdminPassword`