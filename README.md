# turnoff_bluetooth
Script de bash do Linux para quando eu esqueço de desligar o Bluetooth
Se você quiser colocar no crontab, eu sugiro colocar o scrpit no `/bin` e rodar a cada 5 minutos assim:
```
# m h dom mon dow command
*/5 * * * * blue
```

Nesse shell é usado o "gerenciador de bluetooth" `bluetoothctl`

## Walk-through:
``` bash 
if [ $(bluetoothctl show | grep "Powered: yes" -c) -eq 1 ]; then

  CONECTADOS=0
  
...

fi
```
  
Essa condição verifica se o bluetooth está realmente ligado, por que se não estiver não tem nem por que rodar o resto do script.
O ``bluetoothctl show`` me mostra o status do meu dispositivo então eu faço um pipe para um ``grep`` procurando "Powered: yes" e se achar, o parâmetro ``-c`` conta quantos vezes essa expressão aparece, como ela só deve aparecer uma vez, a condição pode ser satisfeita.
Dentro da condição, eu criei uma variável chamada ```CONECTADOS```. ```CONECTAODS``` vai nos ajudar a salvar quais dispositivos estão conectados e dizer se de fato existe algum dispositivo conectado.


Quando a condição é satisfeita rodamos esse bloco de código:

```shell
...

  for (( i = 1; i <= $(bluetoothctl paired-devices | wc | awk '{print $1}'); i++ )); do
    if [[ $(bluetoothctl info $(bluetoothctl paired-devices | awk -v i="$i" 'FNR == i {print $2}') | grep "Connected: yes" -c) -eq 1 ]]; then
      ARRAY[$CONECTADOS]=$i
      ((CONECTADOS++))
    fi
  done
```

Para verificarmos todos os dispositivos que estão pareados com o computador, precisamos de um `for` que varra esses dispostivos. Para descobrir o número de dispositivos usei o comando que mostra todos os dispositivos, mandei para o `wc` por um pipe e peguei usando `awk` a primeira coluna do único retorno do `wc` que é, nada mais nada menos, o número de linhas que é, consequentemente, o número de dispositivos pareados. Agora vamos verificar se dentre esses dispositivos pareados existe algum conectado ao computador, dentro da condição `if` coloquei o comando que me mostra as informações do dispositivo que ele receber como retorno do `awk` que escolhe somente a "máscara" do dispositvo `i` na lista dada por `bluetoothctl paired-devices`, dessas informções o `grep` conta quantas vezes existe a expressão "Connected: yes" que deve exister pelo menos uma vez se existe algum dispositivo conectado. Satisfeita a condição adicionamos qual a linha do dispositivo no `paired-devices` à lista `ARRAY`, depois adicionamos um à variável `CONECTADOS` e aqui acaba tanto esse `if` quanto o `for`.

Saindo do `for` temos outro `if`, vamos verificar a variável `CONECTADOS`:

```shell 
if [[ $CONECTADOS -eq 0 ]]; then
    bluetoothctl power off
    /usr/bin/notify-send "Desligando bluetooth"
    ...
```

Nesse `if` verificamos se `CONECTAODS` é igual a 0 e se sim vamos desligar o bluetooth e mandar uma notificação falando que o bluetooth está sendo desligado.
Caso `CONECTADOS` seja maior que 0, é porque temos algum dispositivo conectado ao computador então vamos mostrar qual dispositivo ou quais dispositivos estão conectados.

```shell
else if [[ $CONECTADOS -gt 1 ]]; then
    echo "Dispositivos Conectados:"
    for element in "${ARRAY[@]}"
    do
      bluetoothctl info $(bluetoothctl paired-devices | awk -v i="$element" 'FNR == i {print $2}') | grep "Name" | awk '{printf("\t%2s %3s\n", $2, $3)}'
    done
    else
      echo "Dispositivo Conectado:"
      bluetoothctl info $(bluetoothctl paired-devices | awk -v i="${ARRAY[0]}" 'FNR == i {print $2}') | grep "Name" | awk '{printf("\t%2s %3s\n", $2, $3)}'
    fi
  fi
```

Aqui o que temos é uma condição só para saber se temos mais que um dispositivo conectado e se sim vamos fazer um `for` para cada elemento de `ARRAY`. Para cada elemento mostramos seu nome (meu fone tem nome e sobrenome, por isso o `$2` e `$3` no `awk`), nada muito diferente do que fizemos no começo do código.

Agora, se tivermos só um dispositivo conectado, só mostramos o seu nome e sobrenome, sem necessidade do `for` e de `ARRAY`.

Por fim se a condição inicial de o bluetooth estar conectado não for satisfeita, o programa mostra na tela que o bluetooth está desligado e não há nada a fazer.
