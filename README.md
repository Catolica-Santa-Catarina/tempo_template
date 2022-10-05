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
