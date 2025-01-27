# lp
from dockerspawner import DockerSpawner
import os, nativeauthenticator
c = get_config()

# Основные настройки JupyterHub
c.JupyterHub.ip = '0.0.0.0'  # Позволяет принимать подключения от всех интерфейсов (jupyterhub)
c.JupyterHub.port = 8000  # Указывает, на каком порту слушать (35001/8000)
# c.JupyterHub.base_url = '/'  # Базовый URL

# URL-адрес для доступа из внешнего мира
# c.JupyterHub.bind_url = 'http://skayfaks.keenetic.pro:35001'

# Назначаем класс аутентификации
c.JupyterHub.authenticator_class = "nativeauthenticator.NativeAuthenticator"
# Настраиваем администратора
admin = os.environ.get("JUPYTERHUB_ADMIN")
if admin:
    c.Authenticator.admin_users = [admin]

# Настраиваем Spawner для использования Docker
c.JupyterHub.spawner_class = "dockerspawner.DockerSpawner"
# Определяем образ Docker, который будет использоваться для каждого пользователя
c.DockerSpawner.image = os.environ["DOCKER_NOTEBOOK_IMAGE"]
# c.DockerSpawner.use_internal_ip = True
c.DockerSpawner.network_name = "jupyterhub-network"

# Имя сети Docker (если требуется)c.DockerSpawner.network_name = 'your-docker-network'Убедитесь, что рабочая директория содержит все нужные ресурсы
notebook_dir = os.environ.get("DOCKER_NOTEBOOK_DIR", "/home/jovyan/work")
c.DockerSpawner.notebook_dir = '/home/jovyan/work'
c.DockerSpawner.volumes = {"jupyterhub-user-{username}": notebook_dir}
# Опция для удаления контейнеров после остановки
c.DockerSpawner.remove = True
c.DockerSpawner.debug = True
# Опционально: установка ресурсоемкости контейнеров, лимиты CPU и памяти
# c.DockerSpawner.cpu_limit = 1
# c.DockerSpawner.mem_limit = '2G'

# Условия для настройки проверки подлинности NativeAuthenticator
c.Authenticator.allowed_failed_logins = 3
# Включаем возможность регистрации пользователей
c.Authenticator.open_signup = True  
# Allow all signed-up users to login
c.Authenticator.allow_all = True

# Persist hub data on volume mounted inside container
c.JupyterHub.cookie_secret_file = "/data/jupyterhub_cookie_secret"
c.JupyterHub.db_url = "sqlite:////data/jupyterhub.sqlite"
# Логирование и отладкиПодробный вывод может быть полезен для диагностики ошибок
c.JupyterHub.log_level = 'DEBUG'



version: "3.8"

services:
  hub:
    build:
      context: .
      dockerfile: Dockerfile.jupyterhub
      # args:
      #   JUPYTERHUB_VERSION: latest
    restart: always
    image: jupyterhub
    container_name: jupyterhub
    networks:
      - jupyterhub-network
    volumes:
      # The JupyterHub configuration file
      - "./jupyterhub_config.py:/srv/jupyterhub/jupyterhub_config.py:ro"
      # Bind Docker socket on the host so we can connect to the daemon from
      # within the container
      - "/var/run/docker.sock:/var/run/docker.sock:rw"
      # # Bind Docker volume on host for JupyterHub database and cookie secrets
      - "jupyterhub-data:/data"
    ports:
      - "35001:8000"
    environment:
      # This username will be a JupyterHub admin
      JUPYTERHUB_ADMIN: admin
      # All containers will join this network
      DOCKER_NETWORK_NAME: jupyterhub-network
      # JupyterHub will spawn this Notebook image for users
      DOCKER_NOTEBOOK_IMAGE: quay.io/jupyter/base-notebook:latest
      # Notebook directory inside user image
      DOCKER_NOTEBOOK_DIR: /home/jovyan/work

volumes:
  jupyterhub-data:

networks:
  jupyterhub-network:
    name: jupyterhub-network



FROM quay.io/jupyterhub/jupyterhub:5.2.1

# Install dockerspawner, nativeauthenticator
# hadolint ignore=DL3013
RUN python3 -m pip install --no-cache-dir \
    dockerspawner \
    jupyterhub-nativeauthenticator

CMD ["jupyterhub", "-f", "/srv/jupyterhub/jupyterhub_config.py"]

[D 2025-01-27 22:40:37.287 JupyterHub dockerspawner:1012] Container 5f7f545 status: {'Dead': False,
     'Error': '',
     'ExitCode': 0,
     'FinishedAt': '0001-01-01T00:00:00Z',
     'Health': {'FailingStreak': 2,
                'Log': [{'End': '2025-01-28T01:40:31.064177168+03:00',
                         'ExitCode': 1,
                         'Output': 'Traceback (most recent call last):\n'
                                   '  File "/etc/jupyter/docker_healthcheck.py", '
                                   'line 27, in <module>\n'
                                   '    json_file = '
                                   'next(runtime_dir.glob("server-.json"))\n'
                                   '                '
                                   '^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^\n'
                                   'StopIteration\n',
                         'Start': '2025-01-28T01:40:30.668069358+03:00'},
                        {'End': '2025-01-28T01:40:34.488714143+03:00',
                         'ExitCode': 1,
                         'Output': 'Traceback (most recent call last):\n'
                                   '  File "/etc/jupyter/docker_healthcheck.py", '
                                   'line 27, in <module>\n'
                                   '    json_file = '
                                   'next(runtime_dir.glob("server-.json"))\n'
                                   '                '
                                   '^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^\n'
                                   'StopIteration\n',
                         'Start': '2025-01-28T01:40:34.076000827+03:00'}],
                'Status': 'starting'},
     'OOMKilled': False,
     'Paused': False,
     'Pid': 35389,
     'Restarting': False,
     'Running': True,
     'StartedAt': '2025-01-27T22:40:27.667932897Z',
     'Status': 'running'}
[W 2025-01-27 22:40:37.288 JupyterHub base:1258] User admin is slow to become responsive (timeout=10

Сообщение из логов указывает на несколько ключевых аспектов, связанных с контейнером и его состоянием:
1. **Состояние контейнера**:
    - Контейнер в настоящее время **запущен** (`Running: True`) и не находится в процессе перезапуска (`Restarting: False`).
    - Статус контейнера - `'running'`, но с предупреждением, так как процесс не завершён.

2. **Проблемы со здоровьем контейнера**:
    - Health-check указывает статус `'starting'`, что означает недавнее обновление или скажение контейнера и что он ещё не готов.
    - **FailingStreak: 2**: контейнер стал вызывать ошибку в ходе двух последних проверок работоспособности.

3. **Ошибки в health-check скрипте**:
    - Оба последних выполнения health-check завершились с **ExitCode 1**, что указывает на ошибку выполнения скрипта.
    - Сообщения об ошибках показывают `StopIteration`, вызванное в `/etc/jupyter/docker_healthcheck.py` на строке 27. Это указывает, что в данном месте в коде ожидался результат, но его не было. Ошибка `StopIteration` может возникнуть, если `runtime_dir.glob("*server-*.json")` не находит никаких файлов, соответствующих данному шаблону.

4. **Замедленная реакция пользователя**:
    - Сообщение `[W 2025-01-27 22:40:37.288 JupyterHub base:1258] User admin is slow to become responsive (timeout=10)` указывает на то, что пользовательское окружение (`admin`) не ответило в течение 10 секундного таймаута. Это может быть связано с проблемами в инициализации контейнера или медленным запуском необходимых сервисов.
На практике, такие сообщения могут быть важны для диагностики проблем с запуском контейнеров, особенно если контейнеры должны начинать инициацию разных сервисов, таких как JupyterHub. Необходимо проверить доступные журналы и файлы конфигурации или скрипты для устранения проблем, выявленных в ходе health-check.



