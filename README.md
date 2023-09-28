# Previsão do Tempo

## Objetivo

O objetivo desta atividade é aprender programação assíncrona em Dart. Vamos trabalhar com a obtenção
da localização do dispositivo (GPS) e rede - busca de dados pela internet.

## Implementação

Para aplicarmos esses conhecimentos, vamos construir um aplicativo de previsão do tempo.

### Código base

Este repositório fornece para você os códigos-base para o desenvolvimento:

* lib
  * screens
    * city_screen.dart - código para a tela de seleção de cidade para obtenção da previsão de tempo.
    * loading_screen.dart - código para a tela de busca de localização pelo GPS.
    * location_screen.dart - código para a "tela principal", que traz a previsão do tempo para a cidade atual.
  * services
    * location.dart - conterá o código para localização via GPS.
    * networking.dart - conterá o código para busca dos dados na api de tempo.
    * weather.dart - contém constantes que serão mostradas para o usuário.
  * utilities
    * constants.dart - contém constantes para padronização de estilos e redução de escrita de código.

Tome um tempo para analisar os arquivos antes de começarmos.

## Código

### Obter localização por vários dispositivos

Começaremos por obter a informação de GPS. Para isso, utilizaremos um plugin do flutter,
chamado geolocator:
https://pub.dev/packages/geolocator

Esse plugin busca a localização, seja no android ou iOS.

Para instalá-lo, abra seu `pubspec.yaml` e procure a linha com `cupertino_icons`.
*Abaixo* dessa linha, inclua: `geolocator: ^9.0.2`.
*ATENÇÃO: mantenha a identação da linha anterior.*

Seu arquivo ficará, nessa região, assim:
```yaml
dependencies:
  flutter:
    sdk: flutter

  cupertino_icons: ^1.0.2
  geolocator: ^9.0.2
```

Após isso, execute o Pub get.

Agora vamos utilizar esse pacote no nosso projeto; por enquanto abre o arquivo
`loading_screen.dart`, impporte o geolocator:

`import 'package:geolocator/geolocator.dart';`

Então, dentro da classe `_LoadingScreenState`, crie um método `void getLocation()`, 
que conterá a linha informada pela documentação, para obtermos a localização atual do dispositivo:

```dart
void getLocation() {
  Position position = await Geolocator.getCurrentPosition(desiredAccuracy: LocationAccuracy.low);
}
```

note que ao entrar esse código, você notará que `await` está marcado como vermelho,
precisamos marcar o método getLocation como `async`.

*await* é uma palavra-chave que indica que o método em questão (getCurrentPosition) está sendo executado
assíncronamente - em segundo plano. Dessa forma, precisamos indicar ao dart que o método getLocation é assíncrono.
Métodos assíncronos servem para executarmos tarefas que possam consumir tempo, principalmente por questões de entrada e saída
de dados, buscas via rede, etc, de forma que o dispositivo não fique **travado** aguardando o fim da execução desses métodos.

Então, altere a assinatura de getLocation: `Future<void> getLocation() async {`.

Depois disso, podemos inserir uma linha para imprimir a posição, para efeito de teste. Logo abaixo
da linha `Position position ....`, adicione a linha `print(position)`.

Por fim, necessitamos pedir autorização ao dispositivo para usarmos o GPS. Para isso, precisamos seguir a 
documentação da biblioteca. Vamos precisar alterar a configuração em dois arquivos XML.
* Android:
  * Abra o arquivo no caminho `android/app/src/main/AndroidManifest.xml`.
  * Logo abaixo da linha: `<manifest xmlns:android="http://schemas.android.com/apk/res/android" package="com.example.tempo_template">`, adicione a linha:
  
  `<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />`
  * Aqui informamos ao android que nosso aplicativo utilizará a permissão de localização "grosseira" (ao contrário da fina, para, por exemplo, o waze)
* iOS:
  * Abra o arquivo no caminho `ios/Runner/Info.plist`
  * Logo abaixo de `<dict>` adicione as linhas:
```xml
  <key>NSLocationWhenInUseUsageDescription</key>
  <string>This app needs access to location when open.</string>
```

Após essa alteração, vamos criar uma nova função no código de `loading_screen.dart`. Essa função deve ser criada
logo acima da função `getLocation` que acabamos de criar:

```dart
  Future<void> checkLocationPermission() async {
    bool serviceEnabled;
    LocationPermission permission;

    serviceEnabled = await Geolocator.isLocationServiceEnabled();
    if (!serviceEnabled) {
      // serviço de localização desabilitado. Não será possível continuar
      return Future.error('O serviço de localização está desabilitado.');
    }
    permission = await Geolocator.checkPermission();
    if (permission == LocationPermission.denied) {
      permission = await Geolocator.requestPermission();
      if (permission == LocationPermission.denied) {
        // Sem permissão para acessar a localização
        return Future.error('Sem permissão para acesso à localização');
      }
    }
    if (permission == LocationPermission.deniedForever) {
      // permissões negadas para sempre
      return Future.error('A permissão para acesso a localização foi negada para sempre. Não é possível pedir permissão.');
    }
  }

```

Agora, *antes* da linha `Position position ...`, em `getLocation`, inclua uma chamada para essa
nova função: `await checkLocationPermission();`.

No fim, a função `getLocation` ficará assim:
```dart
Future<void> getLocation() async {
    // Verificando permissão de acesso
    await checkLocationPermission();

    // agora podemos pedir a localização atual!
    Position position = await Geolocator.getCurrentPosition(desiredAccuracy: LocationAccuracy.low);
    print(position);
  }
```

Execute seu código no emulador. Pressione o botão azul, note que o android solicitará permissão de localização.
Depois, observe a janela de console, Você verá impressa uma latitude e longitude. Esses valores
são configuráveis no emulador. 

Você pode clicar nos "três pontos" sobre a janela do emulador.


Uma nova janela aparecerá. Na aba Location, você verá um mapa. Pode 
navegar com o mouse pelo mapa e selecionar uma localização clicando duas vezes sobre o ponto.


Ali, você pode clicar em save point e armazenar o ponto para um teste futuro.


E você pode clicar em set location. A partir daí seu celular emulado estará enviando essa localização.


### Métodos para o ciclo de vida de um **Stateful Widget**

No momento estamos buscando a localização ao clicar no botão. Mas idealmente, queremos que
essa funcionalidade aconteça ao abrirmos o aplicativo. Para isso, precisamos entender um pouco sobre
o ciclo de vida dos **Widgets** (**Widget Lifecycle*).

#### Stateless Widgets

No caso de um Stateless Widget, O Ciclo de vida é simples. Ele é construído e destruído, quando não for mais
utilizado. Ele não tem estado. Toda mudança (por exemplo uma cor diferente) é implementada com uma destruição e reconstrução.

#### Stateful Widgets

Os Stateful Widgets também podem ser combinados e eles possuem um estado. Sabemos que têm um estado
que pode ser alterado através do método `setState`. Um Stateful Widget tem um tempo de vida maior, e
também tem mais métodos:

```dart
void initState() {
  // é disparado quando o Widget for criado
}

Widget build(BuildContext context) {
  // disparado quando o objeto for construído e o widget aparecerá na tela
  return null;
}

void deactivate() {
  // é disparado quando o widget foi destruído
}
```

### Carregando a localização *sem* que qualquer botão seja pressionado

Primeiro, remova todo o conteúdo *dentro* de `Scaffold`. Vamos tirar o botão de carregamento.

Agora, crie o método initState (ao digitar, um snippet de código aparecerá).
Crie esse método logo acima do método `build`:

```dart
@override
  void initState() {
    super.initState();
    getLocation();
  }
```

Experimente recarregar seu programa. Agora, sem que você pressione qualquer coisa, a posição será recebida.

### Refatorando o código

Idealmente, lógica de negócio, ou de serviços, não deve ser inserida na tela. Vamos refatorar esse código, para isolar lógica
de serviço do "desenho" de tela.

#### Desafio

Refatore o código do aplicativo de forma que a lógica de obter a localização atual seja 
manejada por um objeto `Location`.

* Crie uma classe `Location` no arquivo `lib/services/location.dart`;
* Essa classe deve ter dois atributos: `latitude` e `longitude`. Ambos do tipo `double`.
* Mova o método `checkLocationPermission` para a classe `Location`.
* A classe `Location` também deve ter um método `getCurrentLocation()`. Mova o código de `getLocation()` para o novo método `getCurrentLocation()`.
* O método `getCurrentLocation()` deve fazer com que os valores de `latitude` e `longitude` da `position` sejam atribuídos aos atributos `latitude` e `longitude` da classe `Location`.
* No arquivo `loading_screen.dart` atualize o `getLocation()` de forma que você:
  * Crie um novo objeto de `Location`
  * chame `getCurrentLocation()`
  * Imprima os valores armazenados em `latitude` e `longitude`
* Note que você não precisa criar um construtor para `Location`. Aqui você pode utilizar o ponto de interrogação `?` para indicar que os atributos `latitude` e `longitude` são anuláveis (só serão preenchidos após chamar-se `getCurrentLocation()`).
* Lembre-se de atualizar as importações, em `location.dart` e `loading_screen.dart`.
* Lembre-se também de que o método `getCurrentLocation` deve ser `async` assíncrono.

### Buscando dados de tempo via API REST

O que é uma API? API - **Application Programming Interface** ou Interface de Programação de Aplicação.
Trata-se de um conjunto de comandos, funções, protocolos e objetos que programadores podem usar para criar software ou interagir com um sistema externo (definição da Wikipedia).
Fornece comandos padrão para executar operações padrão, de forma a não precisar escrever código do zero.


Para buscar os dados, precisamos da biblioteca `http`. No arquivo `pubspec.yaml`, abaixo
da linha `geolocator`, acrescente: `http: ^0.13.5`. Não se esqueça de executar o pub get.
(canto superior direito da tela).

Agora, no arquivo `loading_screen.dart`, no começo do arquivo, adicione a linha que importa o pacote http:
`import 'package:http/http.dart' as http;`.

crie, dentro da classe `_LoadingScreenState` um método para baixarmos dados de exemplo da api do openweathermap:

```dart
void getData() async {
  var url = Uri.parse('https://samples.openweathermap.org/data/2.5/weather?lat=35&lon=139&appid=b6907d289e10d714a6e88b30761fae22');
  http.Response response = await http.get(url);

  if (response.statusCode == 200) { // se a requisição foi feita com sucesso
    var data = response.body;
    print(data);  // imprima o resultado
  } else {
    print(response.statusCode);  // senão, imprima o código de erro
  }
}
```

Neste momento, estamos apenas validando a resposta da consulta.
Para chamar o método, adicione a linha `getData()` ao método `build` da classe.

### Fazendo o parse dos dados JSON

Para fazer o parse dos dados JSON que recebemos, vamos usar o
pacote `dart:convert`. Para isso, no início do arquivo, importe-o:
`import 'package:dart:convert`. Esta biblioteca nos fornece o 
método `jsonDecode`. Na linha abaixo de `var data = response.body;`,
acrescente: `var jsonData = jsonDecode(data);`.

Agora `jsonData` corresponde a uma estrutura do tipo 
[Map](https://api.flutter.dev/flutter/dart-core/Map-class.html).
Basicamente um `Map` é uma coleção de pares chave-valor, de modo
que podemos recuperar um valor com sua chave. No nosso caso,
as chaves serão as chaves do json (coord, lon, lat, weather, id, etc...)
e os valores, seus valores correspondentes. 

A sintaxe para acessarnmos um valor dentro de um `Map` é:
`mapa['chave']`. Essa chamada nos traz o valor na chave `chave`, 
dentro de `mapa`. No nosso caso, o mapa está na variável `jsonData`.
As chaves são as diversas chaves do json. Devemos olhar com atenção
para o json gerado. Você pode usar um [Json Beautifier](https://codebeautify.org/jsonviewer)
para facilitar o trabalho. Cole o json gerado no print e observe a coluna
da direita. Neste texto, vamos mostrar separadamente as partes que nos interessam.

Dentro do JSON recebido, queremos obter:
1. O Id da condição do tempo.
2. A cidade obtida via longitude e latitude
3. A temperatura atual.

Esses dados, no Json, estão em (json simplificado, não estão transcritos todos os campos):

```json
{
  "weather": [{
    "id": 800,
    "main": "Clear"
    // ...
  }],
  "main": {
    "temp": 14.43,
    "feels_like": 13.44
    // ...
  },
  // ...
  "timezone": -25200,
  "id": 537480,
  "name": "Mountain View",
  "cod": 200
}
```

Os caminhos (*path*) para os valores que queremos obter são os seguintes:

* Cidade: `name`
* Temperatura: `main.temp`
* Id da condição do tempo: `weather[0].id`

No caso da cidade, o campo `name` é um *filho direto* do objeto JSON. 
A temperatura, o campo `temp` é um filho de `main`, que é filho direto do
objeto JSON. Por último, o `id` está em um objeto, que está dentro de uma
lista (note que `weather` tem um `[]` como filho direto, de modo que 
seria possível ter mais de uma condição de tempo - aqui estamos ignorando essa possibilidade).
Como queremos a primeira condição de tempo (sempre), precisamos acessá-la pelo
primeiro índice da lista, ou seja, `0`.

Assim, dentro do método `getData`, logo depois da linha do
`jsonDecode`, crie três variáveis para obter esses dados:
```dart
var cityName = jsonData['name'];
var temperature = jsonData['main']['temp'];
var weatherCondition = jsonData['weather'][0]['id'];
```

Para verificar se os valores foram corretamente obtidos, você
pode acrescentar uma linha abaixo, imprimindo esses valores:

`print('cidade: $cityName, temperatura: $temperature, condição: $weatherCondition)`

### Utilizando o dado de localização para obter a temperatura

Para utilizar a API de tempo, passando uma localização, você deverá obter
uma *API Key* no [Open Weather Map](https://openweathermap.org).
Para obter a chave, primeiro inscreva-se clicando [aqui](https://home.openweathermap.org/users/sign_up).

Após inscrever-se e habilitar sua conta, na [página do seu profile](https://home.openweathermap.org/)
você encontrará o link [API keys](https://home.openweathermap.org/api_keys). Ali você 
poderá gerar uma chave para sua aplicação. Usaremos esse valor no passo a seguir.

De posse da API Key, dentro da classe `_LoadingScreenState` crie dois *atributos*:

```dart
  late double latitude;
  late double longitude;
```

Dentro do método `getLocation`, faça com que os valores de latitude e longitude sejam
atribuídos a essas variáveis. Faça também com que `getData` seja chamado de dentro desse
método. `getLocation` deverá ficar assim:

```dart
  Future<void> getLocation() async {
    var location = Location();
    await location.getCurrentPosition();

    latitude = location.latitude!;
    longitude = location.longitude!;

    getData();
  }
```

Em `getData`, vamos agora utilizar nossa URL customizada. No início do arquivo `loading_screen.dart`,
logo após os as linhas `import`, crie uma *constante* com sua API key: 

`const apiKey = 'coloque aqui sua api key gerada no passo anterior';`

Agora mude a linha em que há a atribuição da URL, em `getData`. Vamos inserir nessa string
as variáveis `longitude`, `latitude` e `apiKey`. A linha ficará assim:

`var url = Uri.parse('https://api.openweathermap.org/data/2.5/weather?lat=$latitude&lon=$longitude&appid=$apiKey&units=metric');`

Pronto. Execute o código. Deverá funcionar, buscando a coordenada para qual o seu emulador está
configurado. 

### Refatorando o código.

Até aqui atingimos 70% do objetivo da aplicação: obtemos a localização via GPS, com base nos 
dados do GPS, obtemos a informação de clima. Falta usar essa informação para darmos uma resposta
visual para o usuário.

A primeira coisa a fazer é retirar a lógica de busca dos dados do `loading_screen.dart` (que por fim, é uma tela)
e mover para `services/networking.dart`.

No `networking.dart`, vamos trazer as importações de:

```dart
import 'package:http/http.dart' as http;
import 'dart:convert';
```

E acrescentar mais uma, para o caso de erro na execução da requisição:

`import 'dart:io';`

Crie uma classe `NetworkHelper`, que será responsável por baixar os dados e
fazer o parse do JSON para um `Map`. Essa classe terá apenas um método, o `getData`,
que hoje está em `loading_screen.dart`. A classe receberá por parâmetro uma `url`, que será
utilizada para a consulta à API REST. A classe ficará assim:

```dart
import 'package:http/http.dart' as http;
import 'dart:convert';
import 'dart:io';

class NetworkHelper {
  final String url;

  NetworkHelper(this.url);

  Future getData() async {
    var url = Uri.parse(this.url);
    http.Response response = await http.get(url);

    if (response.statusCode == 200) { // se a requisição foi feita com sucesso
      var data = response.body;

      return jsonDecode(data);
    } else {
      // imprime mensagem na saída de erro
      stderr.writeln(response.statusCode);
    }
  }
}
```

Note que aqui o tipo de retorno de `getData` é `Future`. Isso se faz
necessário pois é um método assíncrono, ou seja, faz uma requisição em segundo plano
e retornamos um valor `dynamic` (dinâmico). Experimente passar o mouse sobre
`jsonDecode`. Você verá que ele retorna um valor `dynamic`. Isso signifoca que 
o tipo do valor só é determinado em tempo de execução. Se nosso método retornasse, por exemplo,
uma `String`, ao invés de `Future getData() async`, a assinatura de nosso método
poderia ser `Future<String> getData() async`.

Outra alteração que fizemos foi na cláusula `else` trocar um `print` por um
`stderr.writeln`. Por isso que importamos a biblioteca `dart:io`. Dessa forma,
caso a requisição não tenha um código de retorno 200, que significa "sucesso",
imprimiremos na console de erro o código de retorno obtido, para fins de
depuração. Usar o `stderr.writeln` ou o método `log`, incluído na biblioteca
`dart:developer` é a maneira mais correta de trabalharmos com impressão na console,
ao invés de usarmos o `print`. Para mais detalhes, consulte [este link](https://docs.flutter.dev/testing/code-debugging).

Por fim, em `loading_screen.dart`, faça as seguintes alterações:

1. Faça com que o método `getData` inicialize uma instância de `NetworkHelper` 
e obtenha os dados passando a url da api por parâmetro. Ele deve ficar assim:
```dart
void getData() async {
    NetworkHelper networkHelper = NetworkHelper('https://api.openweathermap.org/'
        'data/2.5/weather?lat=$latitude&lon=$longitude&appid=$apiKey&units=metric');

    var weatherData = await networkHelper.getData();
  }
```

2. Certifique-se de que `getData` está sendo chamado dentro de `getLocation`:
```dart
Future<void> getLocation() async {
    var location = Location();
    await location.getCurrentPosition();

    latitude = location.latitude!;
    longitude = location.longitude!;

    getData();
  }
```

No próximo passo, vamos utilizar esses dados obtidos aqui para passarmos
para a `location_screen`.

### Fazendo a transição entre telas

Antes de passarmos, efetivamente, os dados para a `location_screen`, precisamos fazer a 
transição entre telas. Para isso, usamos o método `push` da classe `Navigator`. 
Esse método faz com que possamos transicionar entre a tela atual e a próxima, passando por parâmetro
possíveis dados de estado e também uma *instância* da classe para a qual queremos transicionar.

Você vai encontrar mais detalhes e exemplos [neste link](https://docs.flutter.dev/cookbook/navigation/navigation-basics).

Para fazer a transição, vamos criar o método `pushToLocationScreen`, dentro da classe `_LoadingScreenState`:

```dart
 void pushToLocationScreen() {
    Navigator.push(context, MaterialPageRoute(builder: (context) {
      return const LocationScreen();
    }));
  }
```

Note que chamamos `Navigator.push`, passando por parâmetro um `MaterialPageRoute`. 
Aqui o que fazemos é indicar que queremos transicionar para uma página que é baseada 
em um *Widget* do tipo *material*. O parâmetro é a página para onde queremos ir, no caso uma
instância de `LocationScreen`.

Você deve incluir uma chamada para este método no final de `getData`. Não é o ideal,
pois estamos incluindo mais uma responsabilidade para `getData`, mas para o momento e,
dada a simplicidade do aplicativo, faremos desta forma.

Se você executar o aplicativo novamente (aperte o botão *play*), ele deverá fazer a transição
de uma tela "preta" para a tela contendo informação do clima.

Como há um tempo para o download e tratamento do dado, o usuário verá essa "tela preta" sem muita
informação. Para darmos uma resposta melhor para o usuário, vamos incluir um *spinner* na `loading_screen`,
para indicar ao usuário que um processamento está ocorrendo.

### Incluindo um spinner na tela de carregamento

Para o *spinner* usaremos a biblioteca [Flutter Spinkit](https://pub.dev/packages/flutter_spinkit).
Para usá-la, novamente, incluímos a dependência no `pubspec.yaml`. Logo abaixo da linha
onde você incluiu a biblioteca `http`, inclua: `flutter_spinkit: ^5.1.0` e rode o `pub get` (canto superior direito da tela).

Agora, em `loading_screen.dart`, faça a importação da biblioteca: `import 'package:flutter_spinkit/flutter_spinkit.dart';`.

No método `build`, vamos substituir o retorno do `Scaffold` por um widget de `spinner`.

Aqui escolhi o *DoubleBounce*, mas você pode escolher outro, se quiser. 

![DoubleBounce](https://raw.githubusercontent.com/ybq/AndroidSpinKit/master/art/DoubleBounce.gif)

Pode ver as opções no [link](https://pub.dev/packages/flutter_spinkit) com a documentação da biblioteca.

Usando o *DoubleBounce*, o método `build` ficará assim:

```dart
  @override
  Widget build(BuildContext context) {
    return const Center(
      child: SpinKitDoubleBounce(
        color: Colors.white,
        size: 100.0,
      ),
    );
  }
```

Execute a aplicação novamente e veja o *spinner* sendo executado na transição entre a 
tela de carregamento e a tela com a informação de clima.

### Transmitindo os dados de clima obtidos para a tela de clima

Para podermos transmitir os dados do clima, primeiramente, na classe `LocationScreen`
devemos criar um atributo. Este deve ser um atributo do tipo `dynamic`,
pois é o tipo retornado pela função `getData`. Assim, crie o atributo:
```dart
final dynamic localWeatherData;
```

Além disso, você deve incluir este atributo no construtor desta classe, que ficará assim:

```dart
const LocationScreen({required this.locationWeather, Key? key}) : super(key: key);
```

Com isso, podemos, a partir de agora, passar o json que foi convertido em `loading_screen` 
para a `location_screen`. Para isso, alteraremos o método `pushToLocationScreen`, que agora 
receberá os dados json e os transmitirá para a próxima tela:

```dart
  void pushToLocationScreen(dynamic weatherData) {
    Navigator.push(context, MaterialPageRoute(builder: (context) {
      return LocationScreen(locationWeather: weatherData);
    }));
  }

```

Note que a única mudança está no parâmetro recebido, que é transmitido para o construtor
de `LocationScreen`, conforme a nossa alteração, no passo anterior.
Além disso, precisamos alterar a chamada do método, em `getData`, passando os dados
para este método: `pushToLocationScreen(weatherData);`.

Parte do problema resolvido. O valor dos dados de clima foi passado para a classe `LocationScreen`,
mas ainda não conseguimos acessá-lo no "estado", na classe `State`. Para fazer isso,
de dentro da classe `_LocationScreenState` conseguimos acessar qualquer atributo da classe
`LocationScreen` através do objeto `widget`.

Assim, dentro da classe `_LocationScreenState`, vamos criar um método `updateUI`. Vamos criar também
quatro atributos, que conterão os dados que necessitamos para mostrar na tela 
as informações do clima da cidade em questão.

O método `updateUI` deve receber por parâmetro os dados de clima (`weatherData`) e
deve atribuir os valores que necessitamos: temperatura, condição climática e cidade
nos atributos de classe criados. 

Atributos para a classe `_LocationScreenState`: 
```dart
  late int temperature;  // o valor, em inteiros, da temperatura
  late String weatherIcon;  // o ícone para a condição climática
  late String cityName;  // o nome da cidade
  late String message;  // Frase para o usuário, de acordo com a temperatura
```

Vamos utilizar também um objeto da classe `WeatherModel` (contido no arquivo `weather.dart`).
Este objeto contém alguns métodos que retornam os emojis que mostramos na tela,
com o tipo de clima, dependendo do valor do id da condição climática.
Retornam também uma frase para mostrar para o usuário, conforme a temperatura.

Para obter esses valores, utilizaremos, respectivamente, os métodos: `getWeatherIcon` e `getMessage`.

Logo abaixo dos atributos, inclua uma variável para o `WeatherModel` e o instancie:

```dart
  WeatherModel weather = WeatherModel();
```

Método `updateUI`:
```dart
void updateUI(dynamic weatherData) {
    setState(() {
      var condition = weatherData['weather'][0]['id'];
      weatherIcon = weather.getWeatherIcon(condition);
      double temp = weatherData['main']['temp'];
      temperature = temp.toInt();
      message = weather.getMessage(temperature);
      cityName = weatherData['name'];
    });
  }
```

No método `updateUI`, extraímos, como visto na seção anterior,
os valores da condição climática, temperatura e nome da cidade.

A partir da condição climática, buscamos o ícone, com o método `getWeatherIcon`.
Convertemos também a temperatura para um valor inteiro, uma vez que o valor 
que recebemos do payload `weatherData` é um double e as casas decimais, no nosso aplicativo,
não nos interessam.

Por fim, obtemos a mensagem de acordo com o valor da temperatura.

Tudo isso está envolvido por um `setState`, pois toda vez que o valor for
recebido/alterado, queremos que as variáveis fiquem marcadas como alteradas 
e a tela seja atualizada.

Agora, precisamos chamar o método `updateUI`. Ele deve ser chamado, inicialmente,
no método `initState`, que é executado quando a tela for criada. Ele ficará assim:

```dart
  @override
  void initState() {
    super.initState();
    updateUI(widget.locationWeather);
  }
```

#### Desafio

Substitua as variáveis: `temperature`, `weatherIcon`, `message` e `cityName` pelos valores
*hardcoded* na `location_screen`. Lembre-se que para interpolar uma variável
dentro de uma `String`, em Dart, você pode usar a sintaxe: `'Texto $variavel'`.


### Refatorando os métodos de localização

Até aqui, conseguimos trazer as informações de localização e tempo e mostrá-las na tela. Agora, na `location_screen`,
gostaríamos de utilizar o botão com uma "seta", posicionado no canto superior esquerdo da tela.
Este botão, no código como `Icons.near_me`, deveria atualizar nossa posição (caso tenhamos nos movido,
por exemplo), buscando novas informações de tempo para a localização atual. Para podermos buscar
os dados de localização e tempo, também na `location_screen`, devemos refatorar o código. Até aqui, 
a busca de dados está na `loading_screen`, que é uma tela e não deveria conter esse tipo de lógica.

Isso é um problema também, pois se tentarmos acessar esses dados de outras telas (como é o nosso caso, aqui)
teremos dificuldade. Aqui criamos um problema clássico de baixa coesão e alto acoplamento. Uma tela
não deve depender da outra para funcionar e, uma tela deve ter apenas funcionalidades de *display* de informações,
não de busca de dados.

Dito isso, vamos mover as funções que criamos para a classe `WeatherModel`, que, dentro da nossa aplicação atual,
seria a mais adequada (ainda assim poderia ser motivo de debate, mas estamos, neste momento, mantendo a aplicação o mais simples possível).

Primeiro, movemos os *imports* para `weather.dart`. Retire as duas linhas a seguir de `loading_screen` e mova-as para `weather.dart`.
```dart
import 'package:tempo_template/services/location.dart';
import 'package:tempo_template/services/networking.dart';
```

Dentro da classe `WeatherModel`, crie um método `getLocationWeather()`, com o seguinte código:
```dart
  Future<dynamic> getLocationWeather() async {
    Location location = Location();
    await location.getCurrentPosition();

    NetworkHelper networkHelper = NetworkHelper(
        '$openWeatherMapURL?lat=${location.latitude}&lon=${location.longitude}'
            '&appid=$apiKey&units=metric');

    var weatherData = await networkHelper.getData();
    return weatherData;
  }
```

Note que agregamos a busca da localização e dos dados de clima neste método. É um método assíncrono
que retorna um objeto de tipo `dynamic`, pois é o nosso retorno de `NetworkHelper::getData()`.

Aqui fizemos uma pequena alteração também na URL, para deixá-la um pouco menor. A parte "fixa" da url 
foi parar numa constante, que precisamos declarar, bem como a api key, que também tem que ser declarada.
Então, ainda no `weather.dart`, logo após as linhas de `import` acrescente:

```dart
const apiKey = 'sua_api_key';  // substitua essa string pela sua api key
const openWeatherMapURL = 'https://api.openweathermap.org/data/2.5/weather';
```

Agora, precisamos utilizar esse método na `loading_screen` e na `location_screen`. Na classe 
`_LoadingScreenState`, remova o método de busca de dados de tempo e substitua por:
```dart
  void getData() async {
    var weatherData = await WeatherModel().getLocationWeather();
    pushToLocationScreen(weatherData);
  }
```

Simplesmente buscamos os dados e depois os enviamos para a tela de *location*.

O `initState` ficará assim:
```dart
  @override
  void initState() {
    super.initState();
    getData();
  }
```

Agora, no `_LocationScreenState`, procure o `TextButton` com o ícone `near_me`. Altere o `onPressed` desse botão
para que fique assim: 
```dart
onPressed: () async {
  var weatherData = await weather.getLocationWeather();
  updateUI(weatherData);
},
```
Note aqui que a função foi marcada como assíncrona, pois ela deve aguardar o térnimo da execução de `getLocationWeather` 
para então chamar `updateUI`. Aqui o que fazemos é: ao pressionar o botão de localização (canto superior esquerdo da tela),
atualizamos a localização do GPS e mostramos a nova informação de clima.

Se você estiver testando o programa no seu celular, isso vai funcionar sem problemas. Porém, se estiver testando o programa 
no emulador, não. Para emular corretamente esse comportamento, abra as configurações do emulador (na janela de emulação, os "três pontinhos" no canto superior direito).
Escolha a aba *Location* ou *Localização*, se estiver em português. Ali, você pode escolher qualquer lugar do mapa e apertar o botão *Set Location*.
Depois, volte ao emulador. Saia do programa (botão redondo do android) e vá para o google maps. Clique no botão de ajustar a localização atual. Agora volte para
o programa e clique no botão do canto superior esquerdo. As informações de tempo devem ser atualizadas.

### Ajustes finais de display

Dependendo do emulador (ou do seu celular) que você estiver utilizando, pode ser que a fonte esteja muito grande, ou muito pequena. Você pode ajustar os tamanhos
no arquivo `constants.dart`. Pode reduzir alguns tamanhos (provavelmente você vai querer reduzir o tamanho do ícone de condição do tempo - `kConditionTextStyle` -
experimente algo como `70`).
