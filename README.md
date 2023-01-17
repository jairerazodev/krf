# fastapicrudtest


+ Crear una nueva carpeta para el proyecto, por ejemplo "my_project" y crear un virtual environment en ella.
+ Instalar FastAPI y los paquetes necesarios para trabajar con bases de datos, como por ejemplo SQLAlchemy.
+ Crear un archivo main.py en la carpeta del proyecto y en él, importar las librerías necesarias.
+ Crear una clase llamada Task, que herede de BaseModel de Pydantic para poder utilizar la validación automática de datos y define los atributos de la tarea (id, title, description, status)
+ Crear una clase llamada TaskList, que herede de BaseModel de Pydantic para poder utilizar la validación automática de datos y define los atributos de la lista de tareas (id, tasks)
+ Crear una función llamada create_task que utilizará el decorador @post para recibir una petición post y crear una nueva tarea en la base de datos.
+ Crear una función llamada read_tasks que utilizará el decorador @get para recibir una petición get y devolver todas las tareas existentes en la base de datos.
+ Crear una función llamada update_task que utilizará el decorador @put para recibir una petición put y actualizar una tarea existente en la base de datos.
+ Crear una función llamada delete_task que utilizará el decorador @delete para recibir una petición delete y eliminar una tarea existente en la base de datos.
+ Crear una función llamada main que utilizará el decorador @app.route para establecer la ruta principal de la aplicación y utilizará el método run() de FastAPI para iniciar el servidor.

### **Crear una nueva carpeta para el proyecto y crear un virtual environment en ella:**

  # Crear la carpeta "my_project"
  !mkdir my_project

  # Moverse a la carpeta creada
  %cd my_project

  # Crear un virtual environment con Python 3
  !python3 -m venv env
  
### **Instalar FastAPI y los paquetes necesarios para trabajar con bases de datos, como por ejemplo SQLAlchemy:**

  # Activar el virtual environment
  !source env/bin/activate

  # Instalar FastAPI y SQLAlchemy
  !pip install fastapi sqlalchemy
  
### **Crear un archivo main.py en la carpeta del proyecto y en él, importar las librerías necesarias:**

  # Crear el archivo main.py
  !touch main.py

  # Abrir el archivo con un editor de texto
  !nano main.py

### **main.py**

  # Importamos las librerias necesarias
  import json
  from typing import List
  from fastapi import FastAPI, HTTPException, Request
  from pydantic import BaseModel
  from redis import Redis

  # Instanciamos el objeto de la app de fastapi
  app = FastAPI()

  # Conectamos con el contenedor de redis usando el host 'redis' y el puerto 6379
  redis = Redis(host='redis', port=6379)

  # Creamos la clase Task que hereda de BaseModel de Pydantic
  class Task(BaseModel):
      id: int
      title: str
      description: str
      status: str

  # Creamos la clase TaskList que hereda de BaseModel de Pydantic
  class TaskList(BaseModel):
      id: int
      tasks: List[Task]

  # Creamos la funcion create_task que se encarga de crear una nueva tarea
  @app.post("/tasks/")
  def create_task(task: Task):
      # Convertimos el objeto task a un diccionario
      task_data = task.dict()
      # Generamos un id para la tarea
      task_data['id'] = redis.incr('global:nextTaskId')
      # guardamos los datos en redis
      redis.hmset(f'task:{task_data["id"]}', task_data)
      # retornamos los datos de la tarea
      return task_data

  # Creamos la funcion read_tasks que se encarga de obtener todas las tareas existentes
  @app.get("/tasks/")
  def read_tasks():
      # obtenemos todas las llaves de las tareas
      task_keys = redis.keys('task:*')
      tasks = []
      # Iteramos por las llaves y obtenemos los valores almacenados en redis
      for key in task_keys:
          task = json.loads(redis.hgetall(key))
          tasks.append(task)
      # retornamos la lista de tareas
      return {"tasks": tasks}

  # Creamos la funcion update_task que se encarga de actualizar una tarea existente
  @app.put("/tasks/{task_id}")
  def update_task(task_id: int, task: Task):
      # Convertimos el objeto task a un diccionario
      task_data = task.dict()
      # Agregamos el id de la tarea al diccionario
      task_data['id'] = task_id
      # actualizamos los datos en redis
      redis.hmset(f'task:{task_id}', task_data)
      # retornamos los datos actualizados de la tarea
      return task_data

  # Creamos la funcion delete_task que se encarga de eliminar una tarea existente
  @app.delete("/tasks/{task_id}")
  def delete_task(task_id: int):
      # eliminamos la tarea de redis
      redis.delete(f'task:{task_id}')
      # retornamos un mensaje de confirmacion
      return {"message": "Task deleted"}

  # Creamos la funcion read_root que se encarga de la ruta principal
  @app.get("/")
  def read_root():
      return {"Hello": "World"}

Si quieres manejar el caso en el que se intenta actualizar una tarea que no existe en la base de datos, puedes utilizar una sentencia try-except para capturar una excepción KeyError y retornar una respuesta de error al cliente.

  try:
      task_data = task.dict()
      task_data['id'] = task_id
      redis.hmset(f'task:{task_id}', task_data)
  except KeyError:
      raise HTTPException(status_code=404, detail="Task not found")
    
Para validar los datos recibidos en las peticiones, puedes utilizar las clases Task y TaskList que ya has creado para definir los esquemas de los datos que esperas recibir. Pydantic se encargará de validar automáticamente los datos recibidos de acuerdo con esos esquemas, y retornará una respuesta de error si los datos no son válidos.

Por otra parte, para realizar pruebas unitarias sobre estas funciones, puedes utilizar la librería Pytest. Un ejemplo de cómo podrías realizar pruebas para la función create_task podría ser el siguiente:


### **Aquí te muestro algunos ejemplos de cómo podrías realizar pruebas para las demás funciones de tu aplicación de tareas usando Pytest:**

  # test_main.py

  def test_read_tasks(client):
      # Crea una tarea de prueba
      task_data = {'title': 'Test Task', 'description': 'This is a test task', 'status': 'pending'}
      client.post("/tasks/", json=task_data)

      # Realiza la petición GET a la ruta '/tasks/'
      response = client.get("/tasks/")
      # Verifica que la respuesta tenga un código de estado 200 (OK)
      assert response.status_code == 200
      # Verifica que la respuesta contenga los datos de la tarea de prueba
      assert response.json() == {"tasks": [task_data]}

  def test_update_task(client):
      # Crea una tarea de prueba
      task_data = {'title': 'Test Task', 'description': 'This is a test task', 'status': 'pending'}
      response = client.post("/tasks/", json=task_data)
      task_id = response.json()['id']

      # Realiza la petición PUT a la ruta '/tasks/{task_id}' con los nuevos datos de la tarea
      new_task_data = {'title': 'Updated Test Task', 'description': 'This is an updated test task', 'status': 'done'}
      response = client.put(f"/tasks/{task_id}", json=new_task_data)
      # Verifica que la respuesta tenga un código de estado 200 (OK)
      assert response.status_code == 200
      # Verifica que la respuesta contenga los nuevos datos de la tarea
      assert response.json() == {"id": task_id, **new_task_data}

  def test_delete_task(client):
      # Crea una tarea de prueba
      task_data = {'title': 'Test Task



      task_data = {'title': 'Test Task', 'description': 'This is a test task', 'status': 'pending'}
      response = client.post("/tasks/", json=task_data)
      task_id = response.json()['id']

      # Realiza la petición DELETE a la ruta '/tasks/{task_id}'
      response = client.delete(f"/tasks/{task_id}")
      # Verifica que la respuesta tenga un código de estado 200 (OK)
      assert response.status_code == 200
      # Verifica que la respuesta contenga el mensaje de confirmación de eliminación de tarea
      assert response.json() == {"message": "Task deleted"}
    
Ten en cuenta que para poder usar la variable client, debes usar un fixture de pytest llamado client, que se encarga de crear un cliente de prueba para realizar las peticiones.

  def test_redis_connection():
      try:
          # Conectamos con el contenedor de redis usando el host 'redis' y el puerto 6379
          redis = Redis(host='redis', port=6379)
          # Insertamos un dato de prueba en la base de datos
          redis.set('test_key', 'test_value')
          # Obtenemos el valor del dato de prueba
          value = redis.get('test_key')
          # Verificamos que el valor obtenido sea el esperado
          assert value == b'test_value'
      except Exception as e:
          # Si se produce una excepción, falla la prueba
          assert False, f'Error de conexion: {e}'
        
De esta manera, si la conexión a la base de datos es exitosa y no se produce ninguna excepción, la prueba se ejecutará sin problemas y se considerará un éxito. Si, por el contrario, se produce una excepción, la prueba fallará y el mensaje de error se mostrará en el resultado de la prueba.
