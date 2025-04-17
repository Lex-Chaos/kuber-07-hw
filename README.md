# Домашнее задание к занятию «Хранение в K8s. Часть 2»
### Цель задания
В тестовой среде Kubernetes нужно создать PV и продемострировать запись и хранение файлов.

------
### Чеклист готовности к домашнему заданию
1. Установленное K8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключенным GitHub-репозиторием.

------
### Дополнительные материалы для выполнения задания
1. [Инструкция по установке NFS в MicroK8S](https://microk8s.io/docs/nfs).
2. [Описание Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).
3. [Описание динамического провижининга](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/).
4. [Описание Multitool](https://github.com/wbitt/Network-MultiTool).

------
### Задание 1
**Что нужно сделать**
Создать Deployment приложения, использующего локальный PV, созданный вручную.
1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool.
2. Создать PV и PVC для подключения папки на локальной ноде, которая будет использована в поде.
3. Продемонстрировать, что multitool может читать файл, в который busybox пишет каждые пять секунд в общей директории.
4. Удалить Deployment и PVC. Продемонстрировать, что после этого произошло с PV. Пояснить, почему.
5. Продемонстрировать, что файл сохранился на локальном диске ноды. Удалить PV. Продемонстрировать что произошло с файлом после удаления PV. Пояснить, почему.
6. Предоставить манифесты, а также скриншоты или вывод необходимых команд.
  
## Ответ:

[Манифест деплоймента](https://github.com/Lex-Chaos/kuber-07-hw/blob/main/files/busybox-multitool-deployment.yml):

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox-multitool-deployment
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox-multitool
  template:
    metadata:
      labels:
        app: busybox-multitool
    spec:
      containers:
      - name: busybox
        image: busybox:latest
        command: 
        - /bin/sh
        - -c
        - |
          while true; do
            echo "$(date +'%Y-%m-%d %H:%M:%S') Test from busybox" >> /tmp/hwstorage/log.txt
            sleep 5
          done
        volumeMounts:
        - name: persistent-storage
          mountPath: /tmp/hwstorage

      - name: multitool
        image: wbitt/network-multitool
        volumeMounts:
        - name: persistent-storage
          mountPath: /tmp/hwstorage
    
      volumes:
      - name: persistent-storage 
        persistentVolumeClaim:
          claimName: pvc-storage         
```


[Манифест pvc](https://github.com/Lex-Chaos/kuber-07-hw/blob/main/files/pvc-homework.yml):

```yml
apiVersion: v1
kind: PersistentVolumeClaim 
metadata:
  name: pvc-storage ##
  namespace: default ##
spec:
  # storageClassName: ""
  volumeMode: Filesystem 
  accessModes: ###
  - ReadWriteOnce ###
  resources: ###
    requests: ###
      storage: 1Gi
```

[Манифест pv](https://github.com/Lex-Chaos/kuber-07-hw/blob/main/files/pv-homework.yml):

```yml
apiVersion: v1
kind: PersistentVolume 
metadata:
  name: pv-homework 
spec:
  capacity: 
    storage: 1Gi ###
  accessModes: ###
    - ReadWriteOnce ###
  hostPath:
    path: /tmp/hw
```

Чтение и запись файла в storage:

![file-storage](https://github.com/Lex-Chaos/kuber-07-hw/blob/main/img/Task1-1.png)

Состояние pv до удаления pvc и deployment:

![conditions](https://github.com/Lex-Chaos/kuber-07-hw/blob/main/img/Task1-2.png)

После удаления pvc и deployment — pv перешло в состояние Released. Файл log.txt сохранился на диске ноды.

После удаления pv файл log.txt сохранился на диске ноды, потому что была установлена опция `persistentVolumeReclaimPolicy: Retain`

![deletion](https://github.com/Lex-Chaos/kuber-07-hw/blob/main/img/Task1-3.png)

------
### Задание 2
**Что нужно сделать**
Создать Deployment приложения, которое может хранить файлы на NFS с динамическим созданием PV.
1. Включить и настроить NFS-сервер на MicroK8S.
2. Создать Deployment приложения состоящего из multitool, и подключить к нему PV, созданный автоматически на сервере NFS.
3. Продемонстрировать возможность чтения и записи файла изнутри пода.
4. Предоставить манифесты, а также скриншоты или вывод необходимых команд.

## Ответ:

Установил согласно инструкции NFC на microk8s.

[Создал storageClass](https://github.com/Lex-Chaos/kuber-07-hw/blob/main/files/sc-nfc.yml):

```yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.203.50
  share: /srv/nfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - hard
  - nfsvers=4.2
```

[Манифест pvc](https://github.com/Lex-Chaos/kuber-07-hw/blob/main/files/pvc-homework-nfc.yml):

```yml
apiVersion: v1
kind: PersistentVolumeClaim 
metadata:
  name: pvc-storage ##
  namespace: default ##
spec:
  storageClassName: nfs-csi
  volumeMode: Filesystem 
  accessModes: ###
  - ReadWriteOnce ###
  resources: ###
    requests: ###
      storage: 1Gi ###
```

Деплоймент оставил из прошлого задания.

Запись-чтение файла:

![Write-Read](https://github.com/Lex-Chaos/kuber-07-hw/blob/main/img/Task2-2.png)

------
### Правила приёма работы
1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl`, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.