# Arch installation

## Making things easy

To make things easy, it is good idea to perform the installation through ssh. To enable this feature, first set up password for **root** account:

```sh
passwd
```

Next step is to connect to the local network. This step is not necessary if you are using ethernet cable, in that case you should be already connected to the internet. If you are using wifi, follow the steps below.

To connect to wireless network use **iwctl** command. First step is to list avalibe devices:
