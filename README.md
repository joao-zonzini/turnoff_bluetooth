# turnoff_bluetooth
Script de bash do Linux para quando eu esqueço de desligar o Bluetooth
Se você quiser colocar no crontab, eu sugiro colocar o scrpit no `/bin` e rodar a cada 5 minutos assim:
```
# m h dom mon dow command
*/5 * * * * blue
```

Eu tenho quase certeza que nem toda distro usa esse tal de ``bluetoothctl``, eu achei ele na minha e é ele que eu tô usando.
O que eu tenho certeza é que é possível adaptar esse meu script para que funcione com qualquer gerenciador de bluetooth.

## Walk-through:
``` bash 
if [ $(bluetoothctl show | grep "Powered: yes" -c) -eq 1 ]; then

...

fi
```

Essa condição verifica se o bluetooth está realmente ligado, por que se não estiver não tem nem por que rodar o resto do script.
O ``bluetoothctl show`` me mostra o status do meu dispositivo então eu faço um pipe para um ``grep`` procurando "Powered: yes" e se achar, o parâmetro ``-c`` conta quantos vezes essa expressão aparece, como ela só deve aparecer uma vez, a condição pode ser satisfeita.

Quando a condição é satisfeita rodamos esse bloco de código:

```shell
...
CONECTADOS=0

for (( i = 1; i <= $(bluetoothctl devices | wc | awk '{print $1}'); i++ )); do
  if [[ $(bluetoothctl info $(bluetoothctl devices | awk -v i="$i" 'FNR == i {print $2}') | grep "Connected: yes" -c) -eq 1 ]]; then
    ((CONECTADOS++))
    fi
  done
```

Para verificarmos todos os dispositivos que estão pareados com o computador, precisamos de um `for` que varra esses dispostivos. Para descobrir o número de dispositivos usei o comando que mostra todos os dispositivos, mandei para o `wc` por um pipe e peguei usando `awk` a primeira coluna do único retorno do `wc` que é, nada mais nada menos, o número de linhas que é, consequentemente, o número de dispositivos pareados. Agora vamos verificar se dentre esses dispositivos pareados existe algum conectado ao computador, dentro da condição `if` coloquei o comando que me mostra as informações do dispositivo que ele receber como retorno do `awk` que escolhe somente a "máscara" do dispositvo `i` na lista dada por `bluetoothctl devices`, dessas informções o `grep` conta quantas vezes existe a expressão "Connected: yes" que deve exister pelo menos uma vez se existe algum dispositivo conectado. Satisfeita a condição adicionamos um à variável `CONECTADOS` e aqui acaba tanto esse `if` quanto o `for`.

Saindo do `for` temos outro `if`, vamos verificar a variável `CONECTADOS`:

```shell 
if [[ $CONECTADOS -eq 0 ]]; then
    bluetoothctl power off
    notify-send "Desligando bluetooth"
  else
    echo "Dispositivo Conectado"
  fi
```

Nesse `if` verificamos se `CONECTAODS` é igual a 0 e se sim vamos desligar o bluetooth e mandar uma notificação falando que o bluetooth está sendo desligado.
Caso `CONECTADOS` seja maior que 0, é porque temos algum dispositivo conectado ao computador então vamos só mostrar que é isso que tá acontecendo.

Por fim se a condição inicial de o bluetooth estar conectado não for satisfeita, o programa mostra na tela que o bluetooth está desligado e não há nada a fazer.
