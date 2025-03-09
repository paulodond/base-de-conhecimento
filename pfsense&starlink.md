Se você tem um link Starlink modalidade Personal e tem necessidade através dele acessar recursos da su empresa de fora esse tutorial irá lhe ajudar sobre isso.

O link da Starlink usa CGNAT, ou seja, a não ser que tenha um pacote profissional não conseguirá fazer PORTFORWARD para disponiblizar serviços da sua emrpesa para fora.

O que vamos fazer aqui é contornar o CGNAT.

## PFSENSE ##

1 - Instalar o pacote WIREGUARD no pfSense

2 - Và em VPN/Wireguard

3 - Clique em ADD TUNNEL

 #Description:#
 Nome do seu túnel, fic a aseu critério.
 
 #Listen Port:# 
 Porta onde as partes vão escutar para estabelecr comunicação, deixe a defautl mesmo.
 
 #Interface Keys:#
 
 #Private Key:# É grada automaticamente ao clicar no botão GENERATE
 
 #Public Key:# Ao clicar em GENERATE será criada uma chave pública que iremos usar lá na nossa instância EC2

![image](https://github.com/user-attachments/assets/f6a78f84-4ba6-4b16-a2d6-6e08b0997d61)

#Interface Configuration (tun_wg1)

#Interface Addresses:#

#Insira o IP da sua interface wireguard#
Ex:  10.10.120.2\32

![image](https://github.com/user-attachments/assets/d3c9fbad-dd0f-41bb-9650-5297d442a8c4)

# SALVAR

#Interface Assignments

Vamos adicionar a interface que acabmos de criar lá no WIREGUARD.

![image](https://github.com/user-attachments/assets/118447d3-7de8-4030-9328-f1e96f9d87f9)

Configurar a Interface

![image](https://github.com/user-attachments/assets/95af05ee-33f0-4892-ae24-ae5f29a8a55d)
![image](https://github.com/user-attachments/assets/af30d864-22d1-4c3b-8cf2-13725ae99189)
![image](https://github.com/user-attachments/assets/f900e363-b79b-4db2-9d06-0db3ff28369a)




