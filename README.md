# DaemonSet для обновления конфигурации kubelet

По мотивам [yc-kubelet-flag-editor](https://github.com/yandex-cloud/yc-architect-solution-library/tree/main/yc-kubelet-flag-editor).

Поскольку, практически для всех параметров kubelet, [в официальной документации](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/) указано, что `This parameter should be set via the config file specified by the kubelet's --config flag`, для модификации config-файла было решено использовать слияние YAML-манифестов вместо аргументов командной строки kubelet, при помощи инструмента [`yq`](https://kislyuk.github.io/yq/).

## Описание

Поды, запущенные при помощи DaemonSet, запускаются на всех воркерах. Скрипт внутри конейнера будет выполнять следующее:

1. Периодически выполняет слияние kubelet config с кастомными конфигами, заданными в ConfigMap `kubelet-custom`.
2. Если конфиг, который находится на воркере, отличается от YAML, полученного в результате слияния, конфиг обновляется.
3. Удаляются файлы, содержащие состояния менеджеров CPU и Memory, перезагружается kubelet.

Кастомные конфигурации kubelet находятся в конфигмапе, заданном в манифесте `cm-custom-kubelet.yaml`.

Каждый из элементов этого конфигмапа должнен иметь имя файла с расширением `.yaml` и быть описан в соответствии со структурой [KubeletConfiguration](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/).

При слиянии результирующей конфигурации файлы будут расположены в соответствии с именованием.
<br>Значения повторяющихся элементов будут перекрывать друг друга:

```
kubelet-config.yaml <-- 01-custom.yaml <-- 02-custom.yaml
```

Пример:

`kubelet-config.yaml`
```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
kubeReserved:
  cpu: 80m
  ephemeral-storage: 39Gi
  memory: 1943Mi
maxPods: 110
```

`01-custom.yaml`
```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
kubeReserved:
  cpu: 500m
maxPods: 150
```

`02-custom.yaml`
```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
maxPods: 110
```

В результирующем YAML-конфиге получим:
```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
kubeReserved:
  cpu: 500m
  ephemeral-storage: 39Gi
  memory: 1943Mi
maxPods: 110
```

## Установка

```
kubectl apply -f namespace.yaml
kubectl apply -f cm-script.yaml
kubectl apply -f cm-custom-kubelet.yaml
kubectl apply -f daemonset.yaml
```
