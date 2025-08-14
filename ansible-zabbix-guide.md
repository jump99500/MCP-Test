# Ansible을 사용한 Linux-Windows간 Zabbix Agent Install & Start

## 요구 조건

### 포트 설정
- **10050 Port Open** (Zabbix Agent Port)
- **10051 Port Open** (Zabbix Active Agent Port)
  - Active Agent Port가 열려있어야 Auto Registration이 가능함
- **5986 Port Open** (Ansible SSH Port)

### WinRM 설정
- Winrm을 통한 통신이기 때문에 ps1 파일을 통해 Winrm의 https 연결이 필요함

## 설치 방법

### 1. Ansible 파일 다운로드

```shell
# git을 통한 다운로드
git clone https://github.com/jump99500/Amano.git

# unzip을 통한 해제
unzip -xvf winansible.zip
```

### 2. Ansible 구조 설명

프로젝트는 다음과 같은 tree 구조로 구성되어 있습니다:

- **inventory**: Host와 Host별 변수를 지정한 파일
- **tasks**: 실행할 작업들을 정의
- **templates**: 설정 파일 템플릿
- **vars**: 변수 정의

## 설정 파일

### inventory 파일
Host와 Host별 변수를 지정해 놓은 파일로, ansible-playbook 실행 시 해당 인벤토리를 참조하여 명령 실행

### tasks/main.yml

```yaml
---
- name: Zabbix install for window
  hosts: windows_2
  gather_facts: yes
  pre_tasks:
    - include_vars: "../vars/main.yml"
  tasks:
  - name: Create zabbix dir
    ansible.windows.win_file:
      path: 'C:\Zabbix'
      state: directory

  - name: Download zabbix-agent.zip
    win_get_url:
      url: '{{ zabbix_agent_url }}'
      dest: C:\Zabbix\zabbix_agent.zip

  - name: Check if the service is installed
    win_shell: Get-Service -Name "Zabbix Agent"
    register: service_status

  - name: Unzip zabbix-agent.zip
    win_unzip:
      src: C:\Zabbix\zabbix_agent.zip
      dest: C:\Zabbix\
    when: service_status.rc != 0

  - name: Make config from template
    template:
      src: '/home/ubuntu/winansible-zabbix/winansible-zabbix/templates/zabbix_agentd.win.conf.j2'
      dest: 'C:\zabbix_agentd.conf'
      backup: yes
      newline_sequence: '\r\n'

  - name: Add firewall rule for 10050/tcp port
    win_firewall_rule:
      name: 'Zabbix-Agent'
      localport: '10050'
      action: allow
      direction: in
      protocol: tcp
      remoteip: '{{ ansible_ip_server }}'
      state: present
      enabled: yes

  - name: Add firewall rule for 10051/tcp port
    win_firewall_rule:
      name: 'Zabbix-Agent-Active'
      localport: '10051'
      action: allow
      direction: in
      protocol: tcp
      remoteip: '{{ ansible_ip_server }}'
      state: present
      enabled: yes

  - name: Zabbix-agent Install
    win_command: CMD /c "C:\Zabbix\bin\zabbix_agentd.exe --config C:\zabbix_agentd.conf --install"
    when: service_status.rc != 0

  - name: Start zabbix-agent
    win_command: CMD /C "C:\Zabbix\bin\zabbix_agentd.exe -c C:\zabbix_agentd.conf --start"
    when: service_status.rc != 0
```

### vars/main.yml

```yaml
---
zabbix_ip_server: '10.10.1.89'
zabbix_agent_url: 'https://d1n7pzsiwnmbjx.cloudfront.net/amano/zabbix_agent-6.4.4-windows-amd64-openssl-ssw.zip'
ansible_ip_server: '10.10.1.22'
```

### templates/zabbix_agentd.win.conf.j2

주요 설정 항목:

```ini
LogFile=c:\Zabbix\zabbix_agentd.log
Server={{ zabbix_ip_server }}
ServerActive={{ zabbix_ip_server }}
Hostname={{ ansible_hostname }}
HostMetadata=SSW_Zabbix_Test
Timeout=25
Include=C:\Zabbix\conf\*.conf
```

## 실행 방법

서버에서 파일을 다운로드한 후, inventory를 참조하여 ansible-playbook 명령을 실행:

```shell
ansible-playbook -i inventory winansible-zabbix/tasks/main.yml
```

## 설정 요약

```plaintext
LogFile=C:\Zabbix\zabbix_agentd.log
Server = Zabbix 서버의 IP 주소
# ListenPort = 10050 기본값을 변경하지 않는 경우 그대로 주석 처리합니다.
ServerActive = Zabbix 서버의 IP
Hostname = domaine.com이 없는 내 서버 이름(server1)
```

## 참고 링크
- 원본 Notion 페이지: https://www.notion.so/d2b1b9c161014b39818d9c532cafbc9c

---
*이 문서는 Notion MCP를 통해 자동으로 GitHub에 동기화되었습니다.*