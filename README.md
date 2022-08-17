# Mindbox-exam
Deployment for k8s cluster

Ключевые моменты:

1.Указал стратегию обновления RollingUpdate для деплоймента. Сначала будет ставиться Под с новой версией приложения, <br>
   и когда она поднимется, кубер убьёт реплику старой версии приложения. Такое действие позволит сделать обновление более плавным и безопасным. 
   
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxUnavailable: 1

2.Указал реквесты и лимиты. Т.к. характеристики нод мне не известны, делал "на глаз". Реквесты - это количество ресурсов занимаемые подом на ноде. <br>
  Лимиты нужны когда приложение в моменте может выдать больше нагрузки, например при запуске приложения. Это даст нам возможность быстрее поднять его и немного <br>
  выйти за реквесты. по QoS у меня получился Burstable т.к. лимиты и реквесты отличаются

    spec:
          containers:
          - name: web-app
            image: web-app
            ports:
            - containerPort: 8080
            resources: 
             requests:
              memory: "150Mi"
              cpu: "150m"
             limits:
              memory: "256Mi"
              cpu: "500m"

3.Пробы нужны для оценки доступности приложения. livenessProbe - будет делать health-чек и в случае если приложение не ответит 200-м или 400-м кодом HTTP оно <br>
  провалит проверку, таких проверок по умолчанию 3 , т.е. при провале всех 3-х контейнер в поде перезапустится спустя 9 сек. <br>
  ReadinessProbe - нужна в случае если поступил большой запрос на 1 из подов и он ушёл его отрабатывать, кубер перенаправит запросы на другие поды.<br>
  StartupProbe - поможет нам защитится от CrashLoop. liveness и readiness пробы не будут отрабатывать пока приложение не запустистся.<br>

        args:
        - /server
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            httpHeaders:
            - name: Custom-Header
              value: Awesome
          initialDelaySeconds: 3
          periodSeconds: 2
        readinessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy
          initialDelaySeconds: 5
          periodSeconds: 5
        startupProbe:
          httpGet:
            path: /healthz
            port: liveness-port
          failureThreshold: 30
          periodSeconds: 10

4.Горизонтальное автомасштабирование подов(HPA). В зависимости от нагрузки в нашем кластере будет от 2 до 5 реплик приложения. В нашем случае если нагрузка <br>
  на 1 под поднимется на 90%, HPA запустит еще 1 реплику приложения.

    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: autoscaler
    spec:
      scaleTargetRef:
        apiVersion: apps/v2
        kind: Deployment
        name: my-deployment
      minReplicas: 2
      maxReplicas: 5
      metrics:
      - type: Resource
        resource:
          name: cpu
          targetAverageUtilization: 90
      - type: Resource
