+++
title = 'Abordagens para uso de Variáveis de Ambiente no Flutter'
date = 2022-09-03
+++

Para aplicações que estão em produção, é essencial o uso de variáveis de ambiente para que se possa trocar entre os vários ambientes(produção, desenvolvimento, stagging) de forma mais fácil.

Porém, existem algumas formas de se fazer isso. E algumas delas podem não ser tão boas assim, induzindo desenvolvedores à erros na hora de trabalhar com as variáveis, ou podem até mesmo expor dados sigilosos da sua aplicação.

Nesse artigo, veremos alguns modos de se fazer o controle das variáveis de ambiente e qual o mais indicado para você utilizar numa aplicação que vai ser enviada para produção.

_OBS: leia todo o artigo antes de fazer qualquer tipo de implementação._

## Um exemplo

Dando um exemplo prático de pode acontecer com muitos no início do aprendizado de Flutter: consumo de um backend por meio de uma API Rest.

Para fazer esse consumo será necessário fazer uma requisição HTTP para essa API a partir de uma URL. E ai que está o ponto, onde guardar essa URL?

A primeira opção que muitos pensariam seria guardar a URL em uma variável em um arquivo qualquer(como por exemplo `constants.dart`).

Porém, fazendo dessa forma, quando for preciso gerar uma versão pra mandar pra produção(para as lojas), vai ser necessário **lembrar** de alterar esse valor pra URL de produção. O que vai resultar em você ter um arquivo semelhante a isso:

```dart
String baseURL = "https://prod.minhaapi.com";
// String baseURL = "https://dev.minhaapi.com";
// String baseURL = "https://192.168.1.100";
```

Em que você fica comentando as URLs possíveis para enquanto está desenvolvendo e para quando for mandar pras lojas.

Com isso, fica óbvio que **em algum momento** você vai esquecer de alterar essa variável e vai mandar uma versão do app apontando pra API de desenvolvimento para as lojas. O que é um enorme problema, pois seus usuários não vão nem ter conta no ambiente de desenvolvimento e, portanto, não conseguirão utilizar o app.

Usando variáveis de ambiente como no exemplo anterior temos outro problema: exposição desses valores.

No caso da URL de uma API Rest, não é tão problemático ela ficar exposta no código. Porém há casos que cada ambiente precisa ter um valor sigiloso pra o funcionamento do app, como credenciais de um banco de dados local, API Key de algum serviço externo, etc.

Desse modo, se alguém conseguisse ter acesso ao seu código fonte, poderia facilmente obter esses valores sigilosos.

## Problemas

Com o exemplo apresentado, temos então **2** principais problemas a resolver:

1. Não ter que lembrar de alterar as variáveis de ambiente
2. Não deixar as variáveis de ambiente expostas internamente e externamente

## Utilizando .env

Uma abordagem bem comum no backend é utilizar um arquivo `.env` que vai guardar as variáveis de ambiente. Essa abordagem pode ser feita no Flutter também por meio do package [`flutter_dotenv`](https://pub.dev/packages/flutter_dotenv).

Com ele pode-se criar um arquivo `.env` no seu projeto para armazenar as variáveis de ambiente.

```
BASE_URL=https://dev.minhaapi.com
API_KEY=b8f91e93-2460-4cd6-b746-d7ff121bf57b
```

E a obtenção dos valores podem ser feitos da seguinte forma:

```dart
await dotenv.load(fileName: ".env");
```

Porém com isso, só mudamos as variáveis do arquivo `constants.dart` para `.env`.

O que podemos fazer é criar um arquivo `.env` para cada ambiente e utilizar o recurso de target da CLI do Flutter para buildar o projeto. Assim, podemos criar 2 arquivos: `.env-prod` e `.env-dev` e criar 2 arquivos main para buildar nosso app.

```dart
// arquivo main.dart
void main() {
  await dotenv.load(fileName: ".env-dev");
}
```

```dart
// arquivo main_prod.dart
void main() {
  await dotenv.load(fileName: ".env-prod");
}
```

Na hora de buildar, basta rodarmos:

```shell
// Para rodar em dev
flutter run -t lib/main.dart
// Para rodar em prod
flutter run -t lib/main_prod.dart
```

Assim, resolvemos o primeiro problema de ter que lembrar de trocar todas as variáveis de ambiente quando formos lançar o app pra produção. Porém, com essa abordagem ainda temos o segundo problema: exposição de dados. Já que as variáveis de todos os ambientes ainda estão disponíveis no código.

Além disso, surge um outro grande problema com a utilização desse package que muitos nem sequer sabem. No processo de configuração do `flutter_dotenv` é pedido que o `.env` seja colocado como um asset no `pubspec.yaml`. E é aí onde mora o perigo.

Todo e qualquer arquivo que for incluido como um asset no `pubspec.yaml` está exposto no android. Se alguém mal intencionado baixar seu app, ele pode facilmente extrair o APK(existem site e apps que fazem isso). E ao extrair o APK, todos os assets podem ser visualizados, incluindo o `.env`. Caso você tenha alguma variável sigilosa no `.env`, ela está literalmente disponível pra qualquer um que baixe o app.

## Utilizando Dart Define

Com o Flutter 1.17, foi incluida um novo argumento na linha de comando chamado `dart-define`, que nos permite passar variáveis de ambiente de uma forma muito melhor.

Para utilizar essa feature, basta especificar o nome da variável e seu valor como no exemplo abaixo:

```
flutter run --dart-define=BASE_URL=https://dev.minhaapi.com --dart-define=API_KEY=b8f91e93-2460-4cd6-b746-d7ff121bf57b
flutter build apk --dart-define=BASE_URL=https://dev.minhaapi.com --dart-define=API_KEY=b8f91e93-2460-4cd6-b746-d7ff121bf57b
```

E para utilizar esses valores no seu código:

```dart
final baseURL = String.fromEnvironment("BASE_URL");
final apiKey = String.fromEnvironment("API_KEY");
```

E pronto! Com isso, suas variáveis não ficarão expostas nem no código e nem na aplicação final(APK).

Desse modo, ainda é fácil de se colocar o projeto em alguma ferramenta de CI/CD, já que basta passar as variáveis de ambiente pela linha de comando mesmo.

Pode-se ver que o comando pra rodar(e buildar) o app fica bem grande. Porém há formas resolver isso para que você não tenha que ficar copiando e colando esse comando enorme toda vez.

Se você usa VS Code, crie na raiz do seu projeto, dentro da pasta `.vscode`, um arquivo chamado `launch.json`, com o seguinte conteúdo:

```json
{
  "configurations": [
    {
      "name": "dev",
      "request": "launch",
      "type": "dart",
      "args": [
        "--dart-define=BASE_URL=https://dev.minhaapi.com",
        "--dart-define=API_KEY=b8f91e93-2460-4cd6-b746-d7ff121bf57b"
      ]
    }
  ]
}
```

Dentro do array, você pode colocar todas as configurações de ambientes que precisa. Porém, lembre de colocar o arquivo `.vscode/launch.json` no `.gitignore`, para que as variáveis de ambiente não fiquem expostas para qualquer um que tiver acesso ao repositório.

Assim, basta apertar F5 para rodar o projeto no VSCode já passando as variáveis de ambiente.

Caso você prefira rodar os comandos no terminal, uma solução para não ficar copiando e colando o comando toda vez seria fazer um script que consome o `launch.json` acima e roda o comando correto de acordo com o ambiente especificado.

Segue abaixo um exemplo de Shell Script que fiz:

```bash
#!/bin/bash

GetDartDefines() {
  argsFromEnv=$(jq ".configurations[] | select(.name==\"$1\") | .args[]" .vscode/launch.json)
  resultString=""
  for i in $argsFromEnv
  do
    resultString+="${i} "
  done
  echo $resultString | tr -d \"
}

envs=$(jq ".configurations[].name" .vscode/launch.json)

for i in $envs
do
  upperCaseEnvName=$(echo $i | tr "[:lower:]" "[:upper:]")
  upperCaseGivenEnv=$(echo $1 | tr "[:lower:]" "[:upper:]")
  if [ \"$upperCaseGivenEnv\" == $upperCaseEnvName ] || [ \"$upperCaseGivenEnv\" == $i ]
  then
    selectedEnv=$upperCaseGivenEnv
  fi
done

if [ -z "$1" ]
then
  echo "It's necessary to provide the desired enviroment"
elif [ -z "$2" ]
then
  echo "It's necessary to provide the desired command(run or build)"
elif [ -z "$selectedEnv" ]
then
  echo "Invalid enviroment!"
elif [ $2 == "run" ]
then
  flutter run $(GetDartDefines "$selectedEnv")
elif [ $2 == "build" ] && [ -z "$3" ]
then
  echo "It's necessary to provide the target platform to build"
elif [ $2 == "build" ]
then
  flutter build $3 $(GetDartDefines "$selectedEnv")
else
  echo "Invalid command!"
fi
```

_OBS: Deixando claro que não possuo grandes conhecimentos em criação de shell script e essa foi a primeira vez que fiz um, então podem haver problemas com o script acima._
