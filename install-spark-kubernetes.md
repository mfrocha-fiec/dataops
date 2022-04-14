# Instalação e configuração do Spark em Kubernetes

## Definições


## Requisitos mínimos

* 3 CPUs e e 4g de memória

## Importante

O mecanimos do Spark em cluster irá criar `pods` para executar o `driver`, e este por sua vez cria outros `pods` para os `executores`. Ao final, os `executores` são limos, mas  o `pod` do driver, permanece ativo, mas sem consumir recursos. É necessário um mecanismo de limpeza destes `pods` [Spark Docs](https://spark.apache.org/docs/latest/running-on-kubernetes.html).

## Preparação

É necessário baixar o [Apache Spark(https://spark.apache.org/downloads.html) já compilado e descompatá-lo ?.

## Segurança do Cluster

TODO : Consultar https://spark.apache.org/docs/latest/security.html

## Referências

* https://datenworks.com/quickstart-como-rodar-apache-spark-no-kubernetes/
* https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/design.md#the-crd-controller