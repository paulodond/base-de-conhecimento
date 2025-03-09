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

