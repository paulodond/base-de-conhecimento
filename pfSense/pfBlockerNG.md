Perfeito! Aqui está o **arquivo completo e formatado em Markdown**, com ícones e estrutura amigável, pronto para você colocar diretamente no arquivo `pfSense/pfBlockerNG.md` no seu GitHub:


# 🛡️ Configuração Completa do pfBlocker-NG Devel

## 🔍 O que é o pfBlocker-NG?

É um pacote adicional do pfSense que melhora a segurança da sua rede.  
Permite bloquear acessos com base em **IPs**, **países** ou **listas de ameaças conhecidas**, ajudando a proteger contra ataques, spams e conexões indesejadas.

---

## 🚀 O que ele faz?

- 🌐 Bloqueio geográfico de IPs (por países).
- 📥 Importação de listas públicas como Spamhaus ou DShield.
- 📋 Criação de *aliases* (grupos de IPs) para regras do firewall.
- 🔄 Sincronização com outros dispositivos pfSense.

---

## ⚙️ Como funciona?

- **Interfaces**: Define onde o tráfego entra e sai (WAN, LAN).
- **Listas de IPs**: Controla acessos usando listas confiáveis.
- **Modos**:
  - 🔒 Deny Both (bloqueio total).
  - 🚫 Deny Inbound (bloqueio entrada).
  - 🚫 Deny Outbound (bloqueio saída).
  - 🟢 Permit (permitir acesso).
  - 📄 Somente Alias (cria listas sem aplicar regras).

### 🛡️ Tipos de listas:
- Spamhaus, DShield, iblocklist, listas por continente, spammers globais, etc.

######################################
# 📥 ETAPA 1 – Instalar pfBlocker-NG
######################################

1. Acesse: **System > Package Manager > Available Packages**  
2. Procure: **pfBlockerNG-devel**  
3. Clique em **Install**.


####################################
# 🛠️ ETAPA 2 – Configuração Inicial
####################################

* Vá em: **Firewall > pfBlockerNG**
* Siga o **Wizard**:

  1. NEXT ➔ NEXT
  2. Configure as interfaces:

     * **Inbound**: WAN
     * **Outbound**: LAN
  3. Defina o **VIP Address** (ex.: `10.10.10.1`)
  4. Configure portas HTTP/HTTPS do bloqueio (padrão: 8081 / 8443)
  5. Ative ou desative IPv6, DNSBL, Whitelist.
  6. Clique em **FINISH**.

## ⚙️ Configuração Recomendada (General)

| ⚙️ Opção               | ✅ Recomendado |
| ---------------------- | ------------- |
| Enable pfBlockerNG     | ✅ Sim        |
| Keep Settings          | ✅ Sim        |
| Frequency (Cron)       | Once a Day    |
| Hour                   | 03:00         |
| Download Failure Limit | 3             |

---

## 📊 Logs (Limites Recomendados)

| Tipo de Log     | Limite |
| --------------- | ------ |
| pfBlockerNG Log | 2000   |
| Unified Log     | 2000   |
| Error Log       | 2000   |
| DNSBL Log       | 4000   |
| DNS Reply Log   | 2000   |

## 📦 IP Configuration

* **De-Duplication**: ✅ Ativado
* **CIDR Aggregation**: ✅ Ativado
* **Suppression**: ❌ Desativado (exceto para exceções específicas)
* **Force Global IP Logging**: Opcional
* **Placeholder IP Address**: Não alterar

## 🌎 GeoIP (MaxMind)

1. Crie uma conta gratuita em:
   [🔗 MaxMind GeoLite2](https://www.maxmind.com/en/geolite2/signup)

2. Insira **License Key** em:
   **Firewall > pfBlockerNG > IP > MaxMind GeoIP Configuration**

3. Configure:

   * MaxMind Account ID
   * MaxMind License Key
   * Atualização automática

4. Use GeoIP para bloquear países e regiões perigosas.

## 📌 IPv4 Rules (Regras IP)

* **Inbound Firewall Rules**: BLOCK ✅
* **Outbound Firewall Rules**: REJECT ✅
* **Floating Rules**: Disabled ❌
* **Kill States**: Disabled ❌ (ativa se quiser derrubar sessões ativas).

## 📥 IP Blocklists Recomendadas (IPv4)

* ✅ CINS\_army\_v4
* ✅ ET\_Block\_v4
* ✅ ET\_Comp\_v4
* ✅ ISC\_Block\_v4
* ✅ Spamhaus\_Drop\_v4

> 🔄 Atualize em **Firewall > pfBlockerNG > Update**

## 📊 GeoIP Summary

* **Top Spammers**: Deny Inbound ✅
* **Proxy and Satellite**: Deny Inbound ✅
* Países específicos: Bloqueie conforme necessidade.

## 📊 IPv4 Reputation

* **pMAXEnable**: ✅ Ativado
* **dMAXEnable**: ✅ Ativado
* **ccblack Action**: Block

## ⚠️ DNSBL (Opcional)

* **Enable DNSBL**: ✅ Ativado (ou ❌ desative para performance máxima).
* **DNSBL Mode**: Unbound Mode ✅
* **Whitelist DNSBL**: Adicione domínios essenciais (bancos, governo, etc.).

### Exemplo de Whitelist:

bcb.gov.br
bb.com.br
bradesco.com.br
gov.br
google.com
microsoft.com
zoom.us


* **Blocked Webpage**: dnsbl\_default.php
* **Resolver Cache**: Enable ✅

## 📡 Logs e Monitoramento

🔍 Para visualizar bloqueios em tempo real:

# Bloqueios de firewall:
tcpdump -n -e -ttt -i pflog0 'pf.reason == "block"'

# Logs DNSBL:
tail -f /var/log/pfblockerng/dnsbl.log

# Logs gerais:
tail -f /var/log/pfblockerng/pfblockerng.log

Se necessário, crie manualmente a estrutura de logs:


mkdir -p /var/log/pfblockerng
chmod 755 /var/log/pfblockerng
chown root:wheel /var/log/pfblockerng
/usr/local/bin/php /usr/local/www/pfblockerng/pfblockerng.php cron

## 🧪 Testes Práticos

* **Teste IP Bloqueado**:

  ping 185.45.188.0


* **Teste DNSBL**:

  nslookup ads2.adnxs.com 192.168.1.1
  

* **Teste Navegador**:
  Acesse sites como:

  * [http://ads2.adnxs.com](http://ads2.adnxs.com)
  * [http://doubleclick.net](http://doubleclick.net)

---

## 🎯 Dica Final

Se algum teste falhar, revise:

* Listas ativas e modo de bloqueio (IP / DNS).
* Configuração correta de interfaces.
* Regras do firewall.
* Logs para diagnósticos.

---

> 📘 Este guia é um resumo prático e otimizado para ambientes corporativos ou domésticos avançados.
