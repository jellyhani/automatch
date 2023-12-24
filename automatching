from datetime import datetime
import json
import subprocess
import time
import sys
from PyQt5 import QtCore, QtGui, QtWidgets
from PyQt5.QtCore import QUrl, QTimer, QThread, pyqtSignal
from PyQt5.QtGui import QDesktopServices
from PyQt5.QtWidgets import QApplication, QLabel, QMessageBox, QPushButton, QSpinBox, QVBoxLayout, QWidget
from bs4 import BeautifulSoup

import pyperclip
import requests
import urllib3
import urllib

process_name = 'LeagueClientUx.exe'
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

class statusThread(QThread):
    status_updated = pyqtSignal(str)
    process_info_updated = pyqtSignal(str, str, str, str, str, str)
    def __init__(self, main_window, proc_search_thread):
        super(statusThread, self).__init__()
        self.main_window = main_window
        self.proc_search_thread = proc_search_thread
        self.proc_search_thread.process_info_updated.connect(self.process_info_updated)
        self.riot_api = ""

    def run(self):
        while True:
            output = subprocess.check_output(f'tasklist /fi "imagename eq {process_name}"', shell=True).decode('iso-8859-1')
            if process_name in output:
                try:
                    Status_url = requests.get(self.riot_api + '/lol-gameflow/v1/gameflow-phase', verify=False)
                    Status_url_response = json.loads(Status_url.text)
                    Status = Status_url_response
                    self.status_updated.emit(Status)
                    QThread.msleep(100)
                    
                except Exception as e:
                    print(f"Error: {e}")
                    self.status_updated.emit(f"Status: {e}")
                    error_message = str(e)
                    pyperclip.copy(error_message)
                except requests.exceptions.RequestException as e:
                    print(f"An error occurred during the request: {e}")
                    self.status_updated.emit(f"Status: {e}")
                    error_message = str(e)
                    pyperclip.copy(error_message)
            else:
                    self.status_updated.emit("Not Connected")
            QThread.msleep(100)
    def process_info_updated(self, client_api, client_token, riot_api, riot_port, riot_token, client_port):
        self.client_api = client_api
        self.client_token = client_token
        self.riot_api = riot_api
        self.riot_port = riot_port
        self.riot_token = riot_token
        self.client_port = client_port

class proc_searchThread(QThread):
    process_info_updated = pyqtSignal(str, str, str, str, str, str)
    def __init__(self, main_window):
        super().__init__()
        self.main_window = main_window
        self.client_api = ""
        self.client_token = ""
        self.riot_api = ""
        self.riot_port = ""
        self.riot_token = ""
        self.client_port = ""
        self.process_name = 'LeagueClientUx.exe'
    def run(self):
        while True:
            try:
                output = subprocess.check_output(f'tasklist /fi "imagename eq {self.process_name}"', shell=True).decode('iso-8859-1')
                if self.process_name in output:
                    command = f'wmic PROCESS WHERE name=\'{self.process_name}\' GET commandline'
                    output = subprocess.check_output(command, shell=True).decode('iso-8859-1')
                    tokens = ["--riotclient-auth-token=", "--riotclient-app-port=", "--remoting-auth-token=", "--app-port="]
                    for token in tokens:
                        value = output.split(token)[1].split()[0].strip('"')
                        if token == "--riotclient-app-port=":
                            self.client_port = value
                        if token == "--riotclient-auth-token=":
                            self.client_token = value
                        if token == "--app-port=":
                            self.riot_port = value
                        if token == "--remoting-auth-token=":
                            self.riot_token = value
                    self.riot_api = f'https://riot:{self.riot_token}@127.0.0.1:{self.riot_port}'
                    self.client_api = f'https://riot:{self.client_token}@127.0.0.1:{self.client_port}'
                    self.process_info_updated.emit(
                        self.client_api, self.client_token,
                        self.riot_api, self.riot_port,
                        self.riot_token, self.client_port,
                    )
                    
                else:
                    self.riot_api = ""
                    self.client_api = ""
                    self.client_token = ""
                    self.client_port = ""
                    self.riot_token = ""
                    self.riot_port = ""
                    self.process_info_updated.emit("", "", "", "", "", "")
            except Exception as e:
                print(f"Error: {e}")
                error_message = str(e)
                pyperclip.copy(error_message)
                self.riot_api = ""
                self.client_api = ""
                self.client_token = ""
                self.client_port = ""
                self.riot_token = ""
                self.riot_port = ""
                self.process_info_updated.emit("", "", "", "", "", "")

class MatchmakingApp(QWidget):
    def __init__(self):
        super().__init__()

        self.match_timer = QTimer(self)
        self.match_timer.timeout.connect(self.matching_timeout)
        self.delay_timer = QTimer(self)
        self.delay_timer.setSingleShot(True)
        self.delay_timer.timeout.connect(self.delay_timer_timeout)
        self.timer_paused_time = 0  # Variable to store paused time
        self.match_start_time = None  # Variable to store the start time
        self.match_timer_duration = 0  # Variable to store the duration set in the spin box
        # Initialize riot_api attribute
        self.riot_api = ""

        self.init_ui()

    def init_ui(self):
        self.setWindowTitle('Matchmaking Program')
        self.status = QLabel('asdf')
        self.match_time_label = QLabel('Matching Time (seconds):')
        self.match_time_spinbox = QSpinBox()
        self.match_time_spinbox.setMaximum(9999)
        self.match_time_spinbox.setValue(300)

        self.start_button = QPushButton('Start Matching')
        self.start_button.clicked.connect(self.start_matching)

        self.cancel_button = QPushButton('Cancel Matching')
        self.cancel_button.clicked.connect(self.cancel_matching)

        layout = QVBoxLayout()
        layout.addWidget(self.status)
        layout.addWidget(self.match_time_label)
        layout.addWidget(self.match_time_spinbox)
        layout.addWidget(self.start_button)
        layout.addWidget(self.cancel_button)


        self.proc_searchThread = proc_searchThread(self)
        self.status_thread = statusThread(self, self.proc_searchThread)
        self.status_thread.status_updated.connect(self.update_status_label)
        self.status_thread.start()
        self.proc_searchThread.process_info_updated.connect(self.update_process_info)
        self.proc_searchThread.start()
        self.match_timer_duration = self.match_time_spinbox.value() * 1000
        self.setLayout(layout)

    def update_status_label(self, status):
        self.status.setText(f"Status: {status}")

    def update_process_info(self, client_api, client_token, riot_api, riot_port, riot_token, client_port):
        self.client_api = client_api
        self.client_token = client_token
        self.riot_api = riot_api
        self.riot_port = riot_port
        self.riot_token = riot_token
        self.client_port = client_port

    def start_matching(self):
        search_url = f'{self.riot_api}/lol-lobby/v2/lobby/matchmaking/search'
        response = requests.post(search_url, verify=False)
        self.match_start_time = datetime.now()
        self.match_timer_duration = self.match_time_spinbox.value() * 1000
        self.match_timer.start(self.match_timer_duration - self.timer_paused_time)
        

    def cancel_matching(self):
        self.match_timer.stop()
        search_url = f'{self.riot_api}/lol-lobby/v2/lobby/matchmaking/search'
        response = requests.delete(search_url, verify=False)
        self.timer_paused_time = 0
        print('Matching canceled')

    def matching_timeout(self):
        output = subprocess.check_output(f'tasklist /fi "imagename eq {process_name}"', shell=True).decode('iso-8859-1')
        if process_name in output:
            Status_url = requests.get(self.riot_api+'/lol-gameflow/v1/gameflow-phase', verify=False)
            Status_url_response = json.loads(Status_url.text)
            Status = Status_url_response
            new_Status = Status
            self.timer_paused_time = self.match_timer.remainingTime()

            elapsed_time = datetime.now() - self.match_start_time
            
            if Status == 'Matchmaking':
                if self.match_timer.isActive() and new_Status == 'ChampSelect':
                    # Timer is already running, pause it
                    self.timer_paused_time = self.match_timer.remainingTime()
                    self.match_timer.start()
                    print(f'Timer paused at {self.timer_paused_time} milliseconds.')
            elif Status == 'ChampSelect':
                print('In champselect state. Stopping the timer briefly.')
                self.match_timer.stop()
                new_Status = Status
            elif Status == 'ReadyCheck':
                print('In champselect state. Stopping the timer briefly.')
                self.match_timer.stop()
            elif Status == 'InProgress':
                self.match_timer.stop()
                self.timer_paused_time = 0
                new_Status = Status
            if elapsed_time.total_seconds() * 1000 >= self.match_timer_duration:
                print('test')
                search_url = f'{self.riot_api}/lol-lobby/v2/lobby/matchmaking/search'
                response = requests.delete(search_url, verify=False)
                self.match_timer.stop()
                self.timer_paused_time = 0

                self.delay_timer.start(10000)  # 10000 milliseconds = 10 seconds

        else:
            pass
    def delay_timer_timeout(self):
        # Your code to be executed after the delay
        search_url = f'{self.riot_api}/lol-lobby/v2/lobby/matchmaking/search'
        response = requests.post(search_url, verify=False)
        self.match_timer.start(self.match_timer_duration - self.timer_paused_time)

if __name__ == '__main__':
    app = QApplication(sys.argv)
    window = MatchmakingApp()
    window.show()
    sys.exit(app.exec_())
