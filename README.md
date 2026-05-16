# -import tkinter as tk
from tkinter import ttk, filedialog, messagebox
import threading
import time
import os
import subprocess
import requests
import pyautogui
import configparser
import shutil
from datetime import datetime
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import base64

class CloudYSHelper:
    def __init__(self, root):
        self.root = root
        self.root.title("云原神自动化助手 v0.1")
        self.root.geometry("800x920")
        self.root.resizable(True, True)

        self.config_file = "config.ini"
        self.running = False
        self.timer_monitor_running = False  # 定时监听开关

        self.images = {
            "start_game": {"name": "开始游戏按钮", "file": "start_game.png", "status": tk.StringVar(value="未上传")},
            "logo": {"name": "原神Logo", "file": "logo.png", "status": tk.StringVar(value="未上传")},
            "queue": {"name": "排队界面", "file": "queue.png", "status": tk.StringVar(value="未上传")}
        }

        self.init_config()
        self.create_widgets()
        self.load_config()
        self.check_image_status()
        self.today_executed = set()  # 记录今日已执行的定时，防止重复跑

    def init_config(self):
        self.config = configparser.ConfigParser()
        self.default_config = {
            "CLOUD_YS_PATH": r"C:\Users\87396\AppData\Local\Programs\miHoYo\GenshinImpactCloudGame\Genshin Impact Cloud Game.exe",
            "WXPUSHER_TOKEN": "",
            "WXPUSHER_UID": "",
            "MAIL_SENDER": "",
            "MAIL_AUTH_CODE": "",
            "MAIL_RECEIVE": "",
            "MAIL_SMTP_SERVER": "smtp.qq.com",
            "MAIL_SMTP_PORT": "465",
            "QW_WEBHOOK": "",
            "TIMER_TIME1": "08:00",
            "TIMER_TIME2": "12:00",
            "TIMER_TIME3": "18:00",
            "START_WAIT": "25",
            "BUTTON_TIMEOUT": "15",
            "NETWORK_WAIT": "30",
            "QUEUE_TIMEOUT": "0",
            "QUEUE_CHECK_INTERVAL": "5",
            "GAME_WAIT": "20",
            "CONFIDENCE": "0.7"
        }

    def create_widgets(self):
        # 微信推送
        config_frame = ttk.LabelFrame(self.root, text="微信推送配置(WxPusher)")
        config_frame.pack(fill=tk.X, padx=10, pady=5)

        ttk.Label(config_frame, text="客户端路径：").grid(row=0, column=0, sticky=tk.W, padx=5, pady=3)
        self.cloud_ys_path_var = tk.StringVar()
        ttk.Entry(config_frame, textvariable=self.cloud_ys_path_var, width=50).grid(row=0, column=1, sticky=tk.EW, padx=5, pady=3)
        ttk.Button(config_frame, text="浏览", command=self.browse_path).grid(row=0, column=2, padx=5, pady=3)

        ttk.Label(config_frame, text="WxPusher Token：").grid(row=1, column=0, sticky=tk.W, padx=5, pady=3)
        self.wxpusher_token_var = tk.StringVar()
        ttk.Entry(config_frame, textvariable=self.wxpusher_token_var, width=50).grid(row=1, column=1, sticky=tk.EW, padx=5, pady=3)

        ttk.Label(config_frame, text="WxPusher UID：").grid(row=2, column=0, sticky=tk.W, padx=5, pady=3)
        self.wxpusher_uid_var = tk.StringVar()
        ttk.Entry(config_frame, textvariable=self.wxpusher_uid_var, width=50).grid(row=2, column=1, sticky=tk.EW, padx=5, pady=3)
        ttk.Button(config_frame, text="测试微信推送", command=self.test_wxpusher).grid(row=2, column=2, padx=5, pady=3)
        config_frame.grid_columnconfigure(1, weight=1)

        # 企业微信
        qw_frame = ttk.LabelFrame(self.root, text="企业微信机器人推送")
        qw_frame.pack(fill=tk.X, padx=10, pady=5)
        ttk.Label(qw_frame, text="Webhook 地址：").grid(row=0, column=0, sticky=tk.W, padx=5, pady=4)
        self.qw_webhook_var = tk.StringVar()
        ttk.Entry(qw_frame, textvariable=self.qw_webhook_var, width=65).grid(row=0, column=1, sticky=tk.EW, padx=5, pady=4)
        ttk.Button(qw_frame, text="测试企业微信", command=self.test_qywechat).grid(row=0, column=2, padx=5, pady=4)
        qw_frame.grid_columnconfigure(1, weight=1)

        # 邮箱
        mail_frame = ttk.LabelFrame(self.root, text="备用邮箱推送配置(QQ邮箱为例)")
        mail_frame.pack(fill=tk.X, padx=10, pady=5)

        ttk.Label(mail_frame, text="发件邮箱：").grid(row=0, column=0, sticky=tk.W, padx=5, pady=3)
        self.mail_sender_var = tk.StringVar()
        ttk.Entry(mail_frame, textvariable=self.mail_sender_var, width=40).grid(row=0, column=1, padx=5, pady=3)

        ttk.Label(mail_frame, text="邮箱授权码：").grid(row=0, column=2, sticky=tk.W, padx=5, pady=3)
        self.mail_auth_var = tk.StringVar()
        ttk.Entry(mail_frame, textvariable=self.mail_auth_var, width=30).grid(row=0, column=3, padx=5, pady=3)

        ttk.Label(mail_frame, text="接收邮箱：").grid(row=1, column=0, sticky=tk.W, padx=5, pady=3)
        self.mail_receive_var = tk.StringVar()
        ttk.Entry(mail_frame, textvariable=self.mail_receive_var, width=40).grid(row=1, column=1, padx=5, pady=3)
        ttk.Button(mail_frame, text="测试邮箱推送", command=self.test_email_push).grid(row=1, column=2, columnspan=2, padx=5, pady=3)

        # 定时时间设置（替换原间隔循环）
        time_frame = ttk.LabelFrame(self.root, text="定点定时执行设置(HH:MM)")
        time_frame.pack(fill=tk.X, padx=10, pady=5)
        ttk.Label(time_frame, text="定时1：").grid(row=0, column=0, padx=5, pady=3)
        self.timer1_var = tk.StringVar()
        ttk.Entry(time_frame, textvariable=self.timer1_var, width=10).grid(row=0, column=1, padx=2)
        ttk.Label(time_frame, text="定时2：").grid(row=0, column=2, padx=5, pady=3)
        self.timer2_var = tk.StringVar()
        ttk.Entry(time_frame, textvariable=self.timer2_var, width=10).grid(row=0, column=3, padx=2)
        ttk.Label(time_frame, text="定时3：").grid(row=0, column=4, padx=5, pady=3)
        self.timer3_var = tk.StringVar()
        ttk.Entry(time_frame, textvariable=self.timer3_var, width=10).grid(row=0, column=5, padx=2)

        # 识图配置
        image_frame = ttk.LabelFrame(self.root, text="识别图片配置")
        image_frame.pack(fill=tk.X, padx=10, pady=5)
        for idx, key in enumerate(["logo", "start_game", "queue"]):
            ttk.Label(image_frame, text=f"{self.images[key]['name']}：").grid(row=idx, column=0, sticky=tk.W, padx=5, pady=3)
            ttk.Label(image_frame, textvariable=self.images[key]['status']).grid(row=idx, column=1, sticky=tk.W, padx=5, pady=3)
            ttk.Button(image_frame, text="上传", command=lambda k=key: self.upload_image(k)).grid(row=idx, column=2, padx=5, pady=3)
        image_frame.grid_columnconfigure(1, weight=1)

        # 高级配置
        adv_frame = ttk.LabelFrame(self.root, text="高级配置")
        adv_frame.pack(fill=tk.X, padx=10, pady=5)
        
        self.start_wait_var = tk.StringVar()
        self.button_timeout_var = tk.StringVar()
        self.network_wait_var = tk.StringVar()
        self.queue_timeout_var = tk.StringVar()
        self.queue_check_interval_var = tk.StringVar()
        self.game_wait_var = tk.StringVar()
        self.confidence_var = tk.StringVar()

        items = [
            ("启动等待秒", self.start_wait_var),
            ("按钮识别超时", self.button_timeout_var),
            ("网络等待秒", self.network_wait_var),
            ("排队超时", self.queue_timeout_var),
            ("检测间隔", self.queue_check_interval_var),
            ("游戏加载等待", self.game_wait_var),
            ("识别置信度", self.confidence_var),
        ]
        for i, (text, var) in enumerate(items):
            row = i // 3
            col = i % 3 * 2
            ttk.Label(adv_frame, text=f"{text}：").grid(row=row, column=col, sticky=tk.W, padx=5, pady=3)
            ttk.Entry(adv_frame, textvariable=var, width=8).grid(row=row, column=col+1, padx=5, pady=3)

        # 运行日志
        log_frame = ttk.LabelFrame(self.root, text="运行日志")
        log_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)
        self.log_text = tk.Text(log_frame, wrap=tk.WORD, state=tk.DISABLED)
        self.log_text.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
        scroll = ttk.Scrollbar(self.log_text, command=self.log_text.yview)
        scroll.pack(side=tk.RIGHT, fill=tk.Y)
        self.log_text.config(yscrollcommand=scroll.set)

        # 功能按钮
        btn_frame = ttk.Frame(self.root)
        btn_frame.pack(fill=tk.X, padx=10, pady=10)
        self.start_btn = ttk.Button(btn_frame, text="立即执行任务", command=self.start_task)
        self.start_btn.pack(side=tk.LEFT, padx=5)
        self.loop_btn = ttk.Button(btn_frame, text="开启定点定时监听", command=self.start_time_monitor)
        self.loop_btn.pack(side=tk.LEFT, padx=5)
        self.stop_btn = ttk.Button(btn_frame, text="停止所有任务", command=self.stop_all, state=tk.DISABLED)
        self.stop_btn.pack(side=tk.LEFT, padx=5)
        ttk.Button(btn_frame, text="保存配置", command=self.save_config).pack(side=tk.RIGHT, padx=5)
        ttk.Button(btn_frame, text="恢复默认", command=self.restore_default).pack(side=tk.RIGHT, padx=5)

    def log(self, msg):
        self.log_text.config(state=tk.NORMAL)
        self.log_text.insert(tk.END, f"[{datetime.now().strftime('%H:%M:%S')}] {msg}\n")
        self.log_text.see(tk.END)
        self.log_text.config(state=tk.DISABLED)

    def browse_path(self):
        p = filedialog.askopenfilename(filetypes=[("exe", "*.exe")])
        if p:
            self.cloud_ys_path_var.set(p)

    def upload_image(self, t):
        f = filedialog.askopenfilename(filetypes=[("图片", "*.png;*.jpg")])
        if f:
            shutil.copy(f, self.images[t]["file"])
            self.check_image_status()
            messagebox.showinfo("提示", "上传成功")

    def check_image_status(self):
        for k in self.images:
            s = "已上传" if os.path.exists(self.images[k]["file"]) else "未上传"
            self.images[k]['status'].set(s)

    # 企业微信推送
    def send_qywechat_msg(self, content):
        webhook = self.qw_webhook_var.get().strip()
        if not webhook:
            return
        try:
            data = {"msgtype": "text", "text": {"content": content}}
            requests.post(webhook, json=data, timeout=10)
            self.log("✅ 企业微信机器人推送完成")
        except Exception as e:
            self.log(f"❌ 企业微信推送失败：{str(e)}")

    def test_qywechat(self):
        webhook = self.qw_webhook_var.get().strip()
        if not webhook:
            messagebox.showerror("错误", "请输入企业微信机器人 Webhook 地址")
            return
        try:
            requests.post(webhook, json={"msgtype":"text","text":{"content":"✅ 云原神助手-企业微信测试成功"}})
            messagebox.showinfo("成功", "企业微信测试消息已发送")
        except Exception as e:
            messagebox.showerror("失败", f"发送失败：{e}")

    # 微信推送
    def test_wxpusher(self):
        token = self.wxpusher_token_var.get().strip()
        uid = self.wxpusher_uid_var.get().strip()
        if not token or not uid:
            messagebox.showerror("错误", "请填写 Token 和 UID")
            return
        try:
            requests.post("https://wxpusher.zjiecode.com/api/send/message", json={
                "appToken": token, "content":"✅ WxPusher微信推送测试成功！",
                "summary":"微信测试消息", "uids":[uid]
            }, timeout=10)
            messagebox.showinfo("成功", "微信测试消息已发送")
        except Exception as e:
            messagebox.showerror("错误", f"微信推送失败：{e}")

    def send_wechat_msg(self, title, content):
        token = self.wxpusher_token_var.get().strip()
        uid = self.wxpusher_uid_var.get().strip()
        if not token or not uid:
            return
        try:
            requests.post("https://wxpusher.zjiecode.com/api/send/message", json={
                "appToken": token, "content":content, "summary":title, "uids":[uid]
            }, timeout=15)
            self.log("✅ 微信推送已完成")
        except:
            self.log("❌ 微信推送异常")

    # 邮箱推送
    def send_email_msg(self, title, content):
        sender = self.mail_sender_var.get().strip()
        auth_code = self.mail_auth_var.get().strip()
        receiver = self.mail_receive_var.get().strip()
        smtp_server = self.default_config["MAIL_SMTP_SERVER"]
        smtp_port = int(self.default_config["MAIL_SMTP_PORT"])
        if not all([sender, auth_code, receiver]):
            return
        try:
            msg = MIMEMultipart()
            sub_encode = base64.b64encode(title.encode("utf-8")).decode("utf-8")
            msg["Subject"] = f"=?UTF-8?B?{sub_encode}?="
            msg["From"] = f"云原神助手 <{sender}>"
            msg["To"] = receiver
            msg.attach(MIMEText(content, "plain", "utf-8"))
            conn = smtplib.SMTP_SSL(smtp_server, smtp_port)
            conn.login(sender, auth_code)
            conn.sendmail(sender, receiver, msg.as_string())
            conn.quit()
            self.log("✅ 备用邮箱推送已完成")
        except Exception as e:
            self.log(f"❌ 邮箱推送失败：{str(e)}")

    def test_email_push(self):
        sender = self.mail_sender_var.get().strip()
        auth_code = self.mail_auth_var.get().strip()
        receiver = self.mail_receive_var.get().strip()
        if not all([sender, auth_code, receiver]):
            messagebox.showerror("错误", "请完整填写发件邮箱、授权码、接收邮箱")
            return
        try:
            self.send_email_msg("【测试】原神助手邮箱推送","✅ 邮箱备用推送测试成功")
            messagebox.showinfo("成功", "测试邮件发送成功")
        except Exception as e:
            messagebox.showerror("失败", f"{e}")

    # 统一推送
    def push_all_result(self, log_content, status):
        status_text = "成功" if status else "失败"
        push_title = f"云原神自动化任务{status_text}"
        push_content = f"【云原神自动化助手】\n状态：{status_text}\n时间：{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n日志：\n{log_content}"
        self.send_wechat_msg(push_title, push_content)
        self.send_email_msg(push_title, push_content)
        self.send_qywechat_msg(push_content)

    # 核心任务逻辑
    def run_task(self):
        log_list = []
        status = True
        try:
            c = {
                "CLOUD_YS_PATH": self.cloud_ys_path_var.get(),
                "START_WAIT": self.start_wait_var.get(),
                "BUTTON_TIMEOUT": self.button_timeout_var.get(),
                "NETWORK_WAIT": self.network_wait_var.get(),
                "QUEUE_TIMEOUT": self.queue_timeout_var.get(),
                "QUEUE_CHECK_INTERVAL": self.queue_check_interval_var.get(),
                "GAME_WAIT": self.game_wait_var.get(),
                "CONFIDENCE": self.confidence_var.get()
            }
            log_list.append("任务开始执行")
            os.system("taskkill /f /im \"Genshin Impact Cloud Game.exe\" 2>nul")
            time.sleep(2)
            subprocess.Popen(c["CLOUD_YS_PATH"])
            time.sleep(int(c["START_WAIT"]))

            logo = pyautogui.locateOnScreen("logo.png", confidence=float(c["CONFIDENCE"]))
            if not logo:
                raise Exception("未检测到原神LOGO")
            wx, wy, ww, wh = logo
            pyautogui.click(wx+50, wy+50)
            time.sleep(1)
            pyautogui.hotkey('win', 'up')
            time.sleep(2)

            btn = pyautogui.locateOnScreen("start_game.png", confidence=float(c["CONFIDENCE"]))
            if btn:
                pyautogui.click(pyautogui.center(btn))

            for i in range(int(c["NETWORK_WAIT"])):
                time.sleep(1)

            start = time.time()
            while True:
                if pyautogui.locateOnScreen("queue.png", confidence=float(c["CONFIDENCE"])):
                    time.sleep(int(c["QUEUE_CHECK_INTERVAL"]))
                    if int(c["QUEUE_TIMEOUT"]) > 0 and time.time()-start > int(c["QUEUE_TIMEOUT"]):
                        raise Exception("排队超时")
                else:
                    break

            time.sleep(int(c["GAME_WAIT"]))
            log_list.append("任务执行完成")
        except Exception as e:
            log_list.append(f"异常：{str(e)}")
            status = False
        finally:
            log_str = "\n".join(log_list[-20:])
            for msg in log_list:
                self.log(msg)
            self.push_all_result(log_str, status)
            self.root.after(0, self.reset_buttons)

    # 定时监听线程
    def time_monitor_worker(self):
        self.log("✅ 定点定时监听已开启，等待到达设定时间自动执行...")
        while self.timer_monitor_running:
            now = datetime.now()
            now_hm = now.strftime("%H:%M")
            today_str = now.strftime("%Y-%m-%d")

            # 每日清空已执行记录
            if not hasattr(self, "last_clear_day") or self.last_clear_day != today_str:
                self.today_executed.clear()
                self.last_clear_day = today_str

            # 获取三个定时
            target_times = [
                self.timer1_var.get().strip(),
                self.timer2_var.get().strip(),
                self.timer3_var.get().strip()
            ]
            for t in target_times:
                if not t:
                    continue
                if now_hm == t and t not in self.today_executed:
                    self.log(f"⏰ 到达设定时间 {t}，自动启动任务")
                    self.today_executed.add(t)
                    threading.Thread(target=self.run_task, daemon=True).start()
            time.sleep(3)

    def start_time_monitor(self):
        self.timer_monitor_running = True
        self.start_btn.config(state=tk.DISABLED)
        self.loop_btn.config(state=tk.DISABLED)
        self.stop_btn.config(state=tk.NORMAL)
        threading.Thread(target=self.time_monitor_worker, daemon=True).start()

    def start_task(self):
        self.start_btn.config(state=tk.DISABLED)
        self.loop_btn.config(state=tk.DISABLED)
        self.stop_btn.config(state=tk.NORMAL)
        threading.Thread(target=self.run_task, daemon=True).start()

    def stop_all(self):
        self.timer_monitor_running = False
        self.running = False
        self.log("🛑 已停止所有定时监听与运行任务")
        self.reset_buttons()

    def reset_buttons(self):
        self.start_btn.config(state=tk.NORMAL)
        self.loop_btn.config(state=tk.NORMAL)
        self.stop_btn.config(state=tk.DISABLED)

    # 配置读写
    def load_config(self):
        if os.path.exists(self.config_file):
            self.config.read(self.config_file, encoding="utf-8")
            s = self.config["Settings"]
            self.cloud_ys_path_var.set(s.get("CLOUD_YS_PATH", self.default_config["CLOUD_YS_PATH"]))
            self.wxpusher_token_var.set(s.get("WXPUSHER_TOKEN", ""))
            self.wxpusher_uid_var.set(s.get("WXPUSHER_UID", ""))
            self.qw_webhook_var.set(s.get("QW_WEBHOOK", ""))
            self.mail_sender_var.set(s.get("MAIL_SENDER", ""))
            self.mail_auth_var.set(s.get("MAIL_AUTH_CODE", ""))
            self.mail_receive_var.set(s.get("MAIL_RECEIVE", ""))
            self.timer1_var.set(s.get("TIMER_TIME1", self.default_config["TIMER_TIME1"]))
            self.timer2_var.set(s.get("TIMER_TIME2", self.default_config["TIMER_TIME2"]))
            self.timer3_var.set(s.get("TIMER_TIME3", self.default_config["TIMER_TIME3"]))
            self.start_wait_var.set(s.get("START_WAIT", self.default_config["START_WAIT"]))
            self.button_timeout_var.set(s.get("BUTTON_TIMEOUT", self.default_config["BUTTON_TIMEOUT"]))
            self.network_wait_var.set(s.get("NETWORK_WAIT", self.default_config["NETWORK_WAIT"]))
            self.queue_timeout_var.set(s.get("QUEUE_TIMEOUT", self.default_config["QUEUE_TIMEOUT"]))
            self.queue_check_interval_var.set(s.get("QUEUE_CHECK_INTERVAL", self.default_config["QUEUE_CHECK_INTERVAL"]))
            self.game_wait_var.set(s.get("GAME_WAIT", self.default_config["GAME_WAIT"]))
            self.confidence_var.set(s.get("CONFIDENCE", self.default_config["CONFIDENCE"]))
        else:
            self.restore_default()

    def save_config(self):
        self.config["Settings"] = {
            "CLOUD_YS_PATH": self.cloud_ys_path_var.get(),
            "WXPUSHER_TOKEN": self.wxpusher_token_var.get(),
            "WXPUSHER_UID": self.wxpusher_uid_var.get(),
            "QW_WEBHOOK": self.qw_webhook_var.get(),
            "MAIL_SENDER": self.mail_sender_var.get(),
            "MAIL_AUTH_CODE": self.mail_auth_var.get(),
            "MAIL_RECEIVE": self.mail_receive_var.get(),
            "TIMER_TIME1": self.timer1_var.get(),
            "TIMER_TIME2": self.timer2_var.get(),
            "TIMER_TIME3": self.timer3_var.get(),
            "START_WAIT": self.start_wait_var.get(),
            "BUTTON_TIMEOUT": self.button_timeout_var.get(),
            "NETWORK_WAIT": self.network_wait_var.get(),
            "QUEUE_TIMEOUT": self.queue_timeout_var.get(),
            "QUEUE_CHECK_INTERVAL": self.queue_check_interval_var.get(),
            "GAME_WAIT": self.game_wait_var.get(),
            "CONFIDENCE": self.confidence_var.get()
        }
        with open(self.config_file, "w", encoding="utf-8") as f:
            self.config.write(f)
        messagebox.showinfo("成功", "配置已保存")

    def restore_default(self):
        self.cloud_ys_path_var.set(self.default_config["CLOUD_YS_PATH"])
        self.wxpusher_token_var.set("")
        self.wxpusher_uid_var.set("")
        self.qw_webhook_var.set("")
        self.mail_sender_var.set("")
        self.mail_auth_var.set("")
        self.mail_receive_var.set("")
        self.timer1_var.set(self.default_config["TIMER_TIME1"])
        self.timer2_var.set(self.default_config["TIMER_TIME2"])
        self.timer3_var.set(self.default_config["TIMER_TIME3"])
        self.start_wait_var.set(self.default_config["START_WAIT"])
        self.button_timeout_var.set(self.default_config["BUTTON_TIMEOUT"])
        self.network_wait_var.set(self.default_config["NETWORK_WAIT"])
        self.queue_timeout_var.set(self.default_config["QUEUE_TIMEOUT"])
        self.queue_check_interval_var.set(self.default_config["QUEUE_CHECK_INTERVAL"])
        self.game_wait_var.set(self.default_config["GAME_WAIT"])
        self.confidence_var.set(self.default_config["CONFIDENCE"])

if __name__ == "__main__":
    root = tk.Tk()
    app = CloudYSHelper(root)
    root.mainloop()
