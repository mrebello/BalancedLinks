# BalancedLinks
Script para balanceamento de carga entre 2 links de acessos no linux.


## instalação
O script de checagem dos links depende do pacote `daemon`:
`apt install daemon`

Copiar os arquivos para os destinos corretos e setar modo para execução:

```
sudo cp balancedlinks_check /sbin
sudo cp balancedlinks_start /sbin
sudo cp balancedlinks_stop /sbin
sudo chmod u+x /sbin/balancedlinks_*
sudo cp balancedlinks.conf /etc
```

para inicialização em systemd (Ubuntu 18+), criar e ativar como serviço no sistema para inicialização automática:

```
sudo cp balancedlinks.service /etc/systemd/system/
systemctl enable balancedlinks
systemctl start balancedlinks
```

## Funcionamento

A tabela de roteamento **main** irá continuar a fazer o roteamento básico de todas as interfaces. As redes acessíveis diretamente pelas interfaces continuam roteadas aqui.

O rotemento padrão (tabela **default**) é excluído.
São criadas 2 tabelas para o roteamento por cada um dos gateways, e uma terceira tabela para o roteamento balanceado entre os dois gateways.<br>
As tabelas usadas foram colocadas com prioridade superior à 32766 (padrão da tabela main).

Pacotes vinculados à interface 1 são roteados pela interface 1 através da **tabela 1**, com prioridade **33001**.
Pacotes vinculados à interface 2 são roteados pela interface 2 através da **tabela 2**, com prioridade **33002**.

Pacotes provenientes do IP da rede da interface 1 são roteados pela interface 1 através da **tabela 1**, com prioridade **33011**.
Pacotes provenientes do IP da rede da interface 2 são roteados pela interface 2 através da **tabela 2**, com prioridade **33002**.

A **tabela 3** (prioridade **33099**) é a responsável pelo balanceamento. Ela roteia por qualquer um dos gateways disponíveis, lenvado em consideração o peso (weight).

O peso *ideal* é proporcional à largura de banda dos links.<br>
Na configuração de exemplo:

link 1 = 100Mbps<br>
link 2 = 20Mbps

Como o link 2 é 5 vezes menor que o link 1, a proporção é 1/5:

weight1=1<br>
weight2=5

### Verificação periódica para queda dos links
O timeout do kernel é relativamente grande para detectar queda nos links, então montei um script para verificar as quedas com frequência maior.


O script **balancedlinks_check** verifica se os gateways podem ser acessados.
Via ping, o acesso aos gateways é feito pela rota padrão da interface.<br>
Via curl, o acesso é vinculado à interface, passando então pelas tabelas com prioridade 33001 e 33002.

Caso um dos links não esteja disponível, o script adiciona a tabela 1 ou 2 com prioridade 33098 (antes da tabela 3), fazendo com que a rota *padrão* passe a ser a rota da interface da tabela adicionada.
Caso os dois links estejam fora, não há rota que resolva...

### Visualização da configuração
`ip route show`<br>
não exibirá mais o 'default'. Ele ficará em outra tabela.

`ip rule show`<br> mostra as regras para o roteamento.

para ver as rotas das tabelas:

```
ip route show table 1
ip route show table 2
ip route show table 3
```

## Testando
O teste pode ser feito checando o IP de origem da saída com o comando

`for ((c=1;c<=20;c++)) do curl 'https://api.ipify.org'; echo; done`

Esse comando exibirá 20 linhas com o ip de origem usando a api do ipify. A proporção entre os links usados deve ser proporcional ao peso (não exata, pois o sistema pode estar fazendo outras conexões simultaneamente).

# ***ATENÇÃO***

Se usa kernel versão >=3.6 e <4.4, dê uma olhada em:
https://www.ti-enxame.com/pt/linux/roteamento-de-caminhos-multiplos-em-kernels-pos-3.6/959920971/
