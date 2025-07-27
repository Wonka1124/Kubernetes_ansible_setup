[English README is available here.](./README_EN.md)





...





# Подробная инструкция по запуску Kubernetes_ansible_setup

## Что это такое?

Этот проект — автоматизированный сценарий (playbook) для Ansible, который сам развернёт Kubernetes-кластер (RKE2) на нескольких серверах.  
Вам не нужно вручную устанавливать Kubernetes — всё сделает Ansible по заранее описанным шагам.

---

## Что потребуется?

- **Компьютер/сервер, с которого вы будете запускать Ansible** (можно обычный Linux, например Ubuntu)
- **Доступ по SSH** к вашим будущим Kubernetes-серверам (root или пользователь с sudo)
- **Python 3.6+** на вашей машине
- **pip** (менеджер пакетов Python)
- **git** (для скачивания репозитория)

---

## Шаг 1. Скачайте проект

```bash
git clone https://github.com/Wonka1124/Kubernetes_ansible_setup
cd Kubernetes_ansible_setup-main
```


---

## Шаг 2. Установите Ansible и зависимости

1. Установите Ansible:
   ```bash
   pip install ansible
   ```
   **Пояснение:**
   - Эта команда установит Ansible через pip. Если pip не установлен: `sudo apt install python3-pip`

   **Пример вывода:**
   ```
   Successfully installed ansible ...
   ```

2. Установите необходимые коллекции для Ansible:
   ```bash
   ansible-galaxy collection install -r collections/requirements.yaml
   ```
   **Пояснение:**
   - Эта команда скачает все нужные модули для работы с Kubernetes и Linux.

   **Пример вывода:**
   ```
   Starting collection install process
   Process install dependency map
   Installing 'ansible.utils:...'
   Installing 'community.general:...'
   Installing 'ansible.posix:...'
   Installing 'kubernetes.core:...'
   ```

---

## Шаг 3. Подготовьте список серверов (inventory)

Откройте файл `inventory/hosts.ini` в любом текстовом редакторе.

Пример:
```
[servers]
server1 ansible_host=192.168.3.21
server2 ansible_host=192.168.3.22
server3 ansible_host=192.168.3.23

[agents]
agent1 ansible_host=192.168.3.24
agent2 ansible_host=192.168.3.25

[rke2:children]
servers
agents

[rke2:vars]
ansible_user=ansible
```

**Пояснение:**
- **servers** — управляющие (master) ноды Kubernetes.
- **agents** — рабочие (worker) ноды.
- **ansible_user** — пользователь, под которым Ansible будет подключаться по SSH (должен существовать на всех серверах и иметь права sudo).

**Проверьте SSH-доступ:**
```bash
ssh ansible@192.168.3.21
```
**Пример вывода:**
```
Welcome to Ubuntu 22.04 LTS ...
ansible@server1:~$
```

---

## Шаг 4. Настройте переменные

Откройте файл `inventory/group_vars/all.yaml` и измените параметры под свою сеть:

```yaml
vip: 192.168.3.50                # Виртуальный IP для доступа к кластеру
metallb_version: v0.13.12        # Версия metallb (балансировщик)
lb_range: 192.168.3.80-192.168.3.90  # Диапазон IP для балансировщика
lb_pool_name: first-pool          # Имя пула для metallb
```

**Пояснение:**
- **vip** — IP-адрес, по которому будет доступен ваш кластер (должен быть свободен в вашей сети).
- **lb_range** — диапазон IP для балансировщика нагрузки (должен быть свободен).

---

## Шаг 5. Проверьте доступность серверов через Ansible

```bash
ansible -i inventory/hosts.ini all -m ping
```
**Пояснение:**
- Эта команда проверит, что Ansible может подключиться ко всем серверам из списка.

**Пример вывода:**
```
server1 | SUCCESS => {"changed": false, "ping": "pong"}
server2 | SUCCESS => {"changed": false, "ping": "pong"}
server3 | SUCCESS => {"changed": false, "ping": "pong"}
agent1 | SUCCESS => {"changed": false, "ping": "pong"}
agent2 | SUCCESS => {"changed": false, "ping": "pong"}
```

Если видите SUCCESS — всё готово!

---

## Шаг 6. Запустите playbook

1. Перейдите в папку с playbook:
   ```bash
   cd Kubernetes_ansible_setup
   ```

2. Запустите основной сценарий:
   ```bash
   ansible-playbook -i inventory/hosts.ini site.yaml
   ```

**Пояснение:**
- Ansible подключится к каждому серверу по SSH и выполнит все шаги по установке и настройке Kubernetes.

**Пример вывода:**
```
PLAY [Prepare all nodes] ****************************************************************
TASK [prepare-nodes : ...] *************************************************************
changed: [server1]
changed: [server2]
changed: [server3]
...
PLAY [Deploy Kube VIP] *****************************************************************
TASK [kube-vip : ...] ******************************************************************
ok: [server1]
ok: [server2]
...
PLAY RECAP ******************************************************************************
server1 : ok=20 changed=15 unreachable=0 failed=0
server2 : ok=20 changed=15 unreachable=0 failed=0
...
```

- **ok=20** — сколько задач выполнено успешно
- **changed=15** — сколько задач что-то изменили
- **failed=0** — ошибок нет

---

## Что будет происходить?

- Ansible подключится к каждому серверу по SSH.
- Подготовит сервера (обновит, установит нужные пакеты).
- Скачает и установит RKE2 (Kubernetes).
- Настроит виртуальный IP (kube-vip).
- Добавит все сервера и агенты в кластер.
- Установит балансировщик нагрузки metallb.

---

## Как проверить, что кластер работает?

1. На управляющем сервере (например, server1) найдите файл kubeconfig:
   ```bash
   sudo cat /etc/rancher/rke2/rke2.yaml
   ```
2. Скопируйте этот файл себе на компьютер и настройте переменную окружения:
   ```bash
   export KUBECONFIG=~/rke2.yaml
   kubectl get nodes
   ```
**Пример вывода:**
```
NAME      STATUS   ROLES                  AGE   VERSION
server1   Ready    control-plane,master   10m   v1.29.4
server2   Ready    control-plane,master   10m   v1.29.4
server3   Ready    control-plane,master   10m   v1.29.4
agent1    Ready    <none>                 8m    v1.29.4
agent2    Ready    <none>                 8m    v1.29.4
```

---

## Важные советы

- **Права пользователя:** Пользователь, под которым Ansible подключается, должен иметь права sudo (или быть root).
- **SSH:** Если требуется пароль, можно добавить флаг `-k` (для пароля пользователя) и `--ask-become-pass` (для sudo).
- **Ошибки:** Если что-то пошло не так — внимательно читайте сообщения об ошибках в консоли. Обычно там понятно, что не так (нет доступа, не хватает прав, не найден пакет и т.д.).
- **Безопасность:** Не используйте root/простые пароли в боевой инфраструктуре!

---

## Часто задаваемые вопросы

**Q: Нужно ли что-то ставить на серверах заранее?**  
A: Нет, только чтобы был доступ по SSH и пользователь с sudo.

**Q: Можно ли запускать с Windows?**  
A: Рекомендуется запускать с Linux или WSL (Windows Subsystem for Linux).

**Q: Как проверить, что кластер работает?**  
A: После завершения playbook на управляющих серверах появится файл kubeconfig (обычно `/etc/rancher/rke2/rke2.yaml`). Можно скопировать его себе и использовать с kubectl.

---

## Полезные команды

- Проверить доступность серверов:
  ```bash
  ansible -i inventory/hosts.ini all -m ping
  ```
- Повторно запустить playbook после исправления ошибок:
  ```bash
  ansible-playbook -i inventory/hosts.ini site.yaml
  ```

---

## Если что-то не получается

- Проверьте SSH-доступ.
- Проверьте права пользователя.
- Проверьте, что IP-адреса и переменные указаны верно.
- Внимательно читайте сообщения об ошибках.



