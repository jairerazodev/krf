# Entorno de desarollo "KRF" Kubernetes, Redis, FastAPI para hacer web scrapping

Un ejemplo de manifiesto para un Pod que contiene una instancia de Redis, una aplicación desarrollada con FastAPI y Python, y un contenedor con Scrapy podría ser el siguiente:

    apiVersion: v1
    kind: Pod
    metadata:
      name: my-pod
    spec:
      containers:
      - name: redis
        image: redis:latest
      - name: web-app
        image: my-web-app-image:latest
        ports:
        - containerPort: 8000
      - name: scraper
        image: my-scraper-image:latest

>Aplicación

Este es un ejemplo sencillo de como se podría ver la aplicación.

    # main.py

    import json
    from typing import List

    from fastapi import FastAPI, HTTPException, Request
    from pydantic import BaseModel
    from redis import Redis

    app = FastAPI()
    redis = Redis(host='redis', port=6379)

    class Task(BaseModel):
        id: int
        title: str
        description: str
        status: str

    class TaskList(BaseModel):
        id: int
        tasks: List[Task]


    @app.post("/tasks/")
    def create_task(task: Task):
        task_data = task.dict()
        task_data['id'] = redis.incr('global:nextTaskId')
        redis.hmset(f'task:{task_data["id"]}', task_data)
        return task_data

    @app.get("/tasks/")
    def read_tasks():
        task_keys = redis.keys('task:*')
        tasks = []
        for key in task_keys:
            task = json.loads(redis.hgetall(key))
            tasks.append(task)
        return {"tasks": tasks}

    @app.put("/tasks/{task_id}")
    def update_task(task_id: int, task: Task):
        task_data = task.dict()
        task_data['id'] = task_id
        redis.hmset(f'task:{task_id}', task_data)
        return task_data

    @app.delete("/tasks/{task_id}")
    def delete_task(task_id: int):
        redis.delete(f'task:{task_id}')
        return {"message": "Task deleted"}

    @app.get("/")
    def read_root():
        return {"Hello": "World"}

>Scrapping

    import aioredis
    from fastapi import FastAPI
    from pydantic import BaseModel
    from typing import List
    from scrapy import Spider
    from scrapy.crawler import CrawlerProcess
    
    app = FastAPI()

    class RequestData(BaseModel):
        text: str

    @app.post("/")
    async def handle_request(request_data: RequestData):
        user_input = request_data.text
        intention = classifier.classify(user_input)

        try:
            # Conectarse a la base de datos Redis en el pod
            
            redis = await aioredis.create_redis(('redis-svc', 6379))

            # Consultar un valor en la base de datos
            
            value = await redis.get("key")
        except Exception as e:
            print(f'Error connecting to redis: {e}')
            return {"response": "Error connecting to the database"}

        # Crear una instancia de scrapy y ejecutarlo
        
        process = CrawlerProcess({
            'USER_AGENT': 'Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 5.1)'
        })
        process.crawl(MySpider, value)
        process.start()

        response = generate_response(intention, value)
        return {"response": response}
    
En este ejemplo, se ha añadido un try except para capturar cualquier posible excepción generada al intentar conectarse al contenedor de Redis y se ha puesto un mensaje de error para el usuario si no se puede conectar. Se ha agregado un ejemplo de uso de scrapy para extraer información de una url.
