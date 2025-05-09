import tkinter as tk
from tkinter import ttk, messagebox
import random
import time
import requests

class QuizApp:
    def __init__(self, master):
        self.master = master
        master.title("Христианская викторина")
        self.style = ttk.Style()
        self.style.configure('TButton', font=('Arial', 16))
        self.style.configure('TLabel', font=('Arial', 16))
        self.style.configure('TEntry', font=('Arial', 16))

        self.min_players = 2
        self.max_players = 5
        self.num_players = self.min_players
        self.player_entries = []
        self.player_names = []
        self.scores = {}
        self.current_player_index = 0
        self.round = 1
        self.max_rounds = 3
        self.team_mode = False
        self.teams = [[], []]
        self.team_scores = [0, 0]
        self.team_names = ["Команда 1", "Команда 2"]
        self.marathon_mode = False
        self.marathon_lives = 1

        self.topics = ["Библия", "Иисус", "Апостолы", "Праздники", "Чудеса"]
        self.levels = [100, 200, 300, 400, 500]

        self.questions = [
            ("Библия", 100, "Сколько книг в Ветхом Завете?", "39"),
            ("Библия", 500, "Кто написал книгу Откровение?", "Иоанн"),
            ("Библия", 200, "Кто вывел израильтян из Египта?", "Моисей"),
            ("Библия", 300, "Кто был самым сильным человеком в Библии?", "Самсон"),
            ("Библия", 400, "Кто был женой Авраама?", "Сарра"),
            ("Иисус", 100, "Где родился Иисус?", "Вифлеем"),
            ("Иисус", 500, "Сколько дней Иисус постился в пустыне?", "40"),
            ("Иисус", 200, "Кто крестил Иисуса?", "Иоанн Креститель"),
            ("Иисус", 300, "Кто был земным отцом Иисуса?", "Иосиф"),
            ("Иисус", 400, "В каком городе Иисус был распят?", "Иерусалим"),
            ("Апостолы", 100, "Сколько было апостолов?", "12"),
            ("Апостолы", 500, "Кто был апостолом для язычников?", "Павел"),
            ("Апостолы", 200, "Кто был братом Петра?", "Андрей"),
            ("Апостолы", 300, "Кто был заменён Матфием?", "Иуда Искариот"),
            ("Апостолы", 400, "Кто был первым мучеником?", "Стефан"),
            ("Праздники", 100, "Какой праздник отмечает воскресение Иисуса?", "Пасха"),
            ("Праздники", 500, "Какой праздник отмечается на 50-й день после Пасхи?", "Пятидесятница"),
            ("Праздники", 200, "Какой праздник отмечает рождение Иисуса?", "Рождество"),
            ("Праздники", 300, "Какой праздник отмечает сошествие Святого Духа?", "Троица"),
            ("Праздники", 400, "Какой праздник отмечает вход Иисуса в Иерусалим?", "Вербное воскресенье"),
            ("Чудеса", 100, "Какое чудо Иисус совершил на свадьбе?", "Превратил воду в вино"),
            ("Чудеса", 500, "Кто был воскрешён Иисусом из мёртвых?", "Лазарь"),
            ("Чудеса", 200, "Сколько хлебов Иисус умножил для народа?", "5"),
            ("Чудеса", 300, "Кто коснулся края одежды Иисуса и исцелился?", "Женщина с кровотечением"),
            ("Чудеса", 400, "Кто ходил по воде вместе с Иисусом?", "Пётр"),
        ]
        self.used_questions = set()

        self.info_label = ttk.Label(master, text="")
        self.topic_var = tk.StringVar(value=self.topics[0])
        self.level_var = tk.IntVar(value=self.levels[0])
        self.topic_menu = ttk.OptionMenu(master, self.topic_var, *self.topics)
        self.level_menu = ttk.OptionMenu(master, self.level_var, *self.levels)
        self.start_button = ttk.Button(master, text="Выбрать тему и уровень", command=self.start_round)
        self.question_label = ttk.Label(master, text="")
        self.answer_entry = ttk.Entry(master)
        self.submit_button = ttk.Button(master, text="Ответить", command=self.submit_answer)
        self.result_label = ttk.Label(master, text="")
        self.new_game_button = ttk.Button(master, text="Новая игра", command=self.restart_game)
        self.load_questions_button = ttk.Button(master, text="Загрузить вопросы из интернета", command=self.load_questions_online)
        self.marathon_button = ttk.Button(master, text="Режим Марафон", command=self.start_marathon)

        self.player_times = {}
        self.current_question = None
        self.current_answer = None
        self.start_time = None
        self.answered_players = []
        self.answers_this_round = []

        self.show_mode_selection()

    def show_mode_selection(self):
        self.clear_widgets()
        ttk.Label(self.master, text="Выберите режим игры:").pack(pady=10)
        ttk.Button(self.master, text="Обычная игра", command=self.show_team_choice).pack(fill=tk.X, padx=20, pady=5)
        self.marathon_button.pack(fill=tk.X, padx=20, pady=5)
        self.load_questions_button.pack(fill=tk.X, padx=20, pady=5)

    def show_team_choice(self):
        self.marathon_mode = False
        self.clear_widgets()
        ttk.Label(self.master, text="Выберите тип игры:").pack(pady=10)
        ttk.Button(self.master, text="Индивидуальная", command=lambda: self.set_team_mode(False)).pack(fill=tk.X, padx=20, pady=5)
        ttk.Button(self.master, text="Командная (2 команды)", command=lambda: self.set_team_mode(True)).pack(fill=tk.X, padx=20, pady=5)

    def set_team_mode(self, team_mode):
        self.team_mode = team_mode
        self.show_num_players_input()

    def show_num_players_input(self):
        self.clear_widgets()
        ttk.Label(self.master, text="Выберите количество игроков (2-5):").pack()
        self.num_players_var = tk.IntVar(value=self.min_players)
        for i in range(self.min_players, self.max_players + 1):
            ttk.Radiobutton(self.master, text=str(i), variable=self.num_players_var, value=i).pack(anchor=tk.W)
        ttk.Button(self.master, text="Далее", command=self.show_name_input).pack()

    def show_name_input(self):
        self.num_players = self.num_players_var.get()
        self.clear_widgets()
        ttk.Label(self.master, text=f"Введите имена {self.num_players} игроков:").pack()
        self.player_entries = []
        for i in range(self.num_players):
            frame = ttk.Frame(self.master)
            frame.pack()
            ttk.Label(frame, text=f"Игрок {i+1}:").pack(side=tk.LEFT)
            entry = ttk.Entry(frame)
            entry.pack(side=tk.LEFT)
            self.player_entries.append(entry)
        ttk.Button(self.master, text="Начать игру", command=self.save_names).pack()

    def save_names(self):
        names = [entry.get().strip() for entry in self.player_entries]
        if any(not name for name in names):
            messagebox.showwarning("Ошибка", "Пожалуйста, введите имена всех игроков!")
            return
        if len(set(names)) < len(names):
            messagebox.showwarning("Ошибка", "Имена игроков должны быть уникальными!")
            return
        self.player_names = names
        self.scores = {name: 0 for name in self.player_names}
        self.current_player_index = 0
        self.round = 1
        self.used_questions = set()
        if self.team_mode:
            self.teams = [[], []]
            for i, name in enumerate(self.player_names):
                self.teams[i % 2].append(name)
            self.team_scores = [0, 0]
        self.update_info()
        self.show_topic_selection()

    def show_topic_selection(self, chooser=None):
        self.clear_widgets()
        self.info_label.pack()
        self.topic_var.set(self.topics[0])
        self.level_var.set(self.levels[0])
        self.topic_menu.pack()
        self.level_menu.pack()
        self.start_button.pack()
        if chooser:
            self.info_label.config(text=f"{chooser}, выберите тему и уровень для следующего раунда.\n" + self.info_label.cget("text"))

    def update_info(self):
        info = f"Раунд {self.round}/{self.max_rounds}\n"
        if self.team_mode:
            info += f"{self.team_names[0]}: {self.team_scores[0]} | {self.team_names[1]}: {self.team_scores[1]}\n"
            info += f"Состав команд:\n{self.team_names[0]}: {', '.join(self.teams[0])}\n{self.team_names[1]}: {', '.join(self.teams[1])}\n"
        else:
            info += "Счёт:\n" + "\n".join([f"{p}: {self.scores[p]}" for p in self.player_names])
        self.info_label.config(text=info)

    def start_round(self):
        self.answered_players = []
        self.player_times = {}
        self.answers_this_round = []
        random.shuffle(self.player_names)
        self.current_player_index = 0
        self.topic_menu.pack_forget()
        self.level_menu.pack_forget()
        self.start_button.pack_forget()
        self.result_label.config(text="")
        self.ask_question()

    def ask_question(self):
        topic = self.topic_var.get()
        level = self.level_var.get()
        available = [
            (i, q, a) for i, (t, l, q, a) in enumerate(self.questions)
            if t == topic and l == level and i not in self.used_questions
        ]
        if not available:
            messagebox.showwarning("Нет вопросов", "Нет доступных вопросов для выбранной темы и уровня.")
            self.end_game()
            return
        idx, question, answer = random.choice(available)
        self.used_questions.add(idx)
        self.current_question = question
        self.current_answer = answer
        self.question_label.config(text=f"{self.player_names[self.current_player_index]}, ваш вопрос:\n{question}")
        self.question_label.pack(pady=10)
        self.answer_entry.pack(pady=5)
        self.answer_entry.delete(0, tk.END)
        self.answer_entry.focus_set()
        self.submit_button.pack(pady=5)
        self.result_label.pack()
        self.start_time = time.time()

    def submit_answer(self):
        answer = self.answer_entry.get().strip()
        player = self.player_names[self.current_player_index]
        elapsed = time.time() - self.start_time
        if not answer:
            messagebox.showwarning("Внимание", "Пожалуйста, введите ответ!")
            return
        self.player_times[player] = (answer, elapsed)
        self.answers_this_round.append((player, answer, elapsed))
        self.answered_players.append(player)
        self.current_player_index += 1
        if self.current_player_index < len(self.player_names):
            self.question_label.config(text=f"{self.player_names[self.current_player_index]}, ваш вопрос:\n{self.current_question}")
            self.answer_entry.delete(0, tk.END)
            self.answer_entry.focus_set()
            self.start_time = time.time()
        else:
            self.evaluate_answers()

    def evaluate_answers(self):
        correct_players = [
            (player, t) for player, (ans, t) in self.player_times.items()
            if ans.lower() == self.current_answer.lower()
        ]
        if correct_players:
            winner = min(correct_players, key=lambda x: x[1])[0]
            if self.team_mode:
                team_idx = 0 if winner in self.teams[0] else 1
                self.team_scores[team_idx] += 1
            else:
                self.scores[winner] += 1
            self.result_label.config(text=f"Правильный ответ: {self.current_answer}\nБыстрее всех ответил {winner}!")
            chooser = winner
        else:
            self.result_label.config(text=f"Никто не ответил правильно! Ответ: {self.current_answer}\nСледующую тему выбирает {self.player_names[0]}.")
            chooser = self.player_names[0]
        self.update_info()
        self.master.after(2500, lambda: self.show_round_summary(chooser))

    def show_round_summary(self, chooser):
        self.clear_widgets()
        summary = ttk.Label(self.master, text="Результаты раунда:")
        summary.pack()
        table = tk.Text(self.master, height=len(self.player_names)+2, width=60, font=('Arial', 14))
        table.insert(tk.END, f"{'Игрок':<15}{'Ответ':<30}{'Время (сек)':<15}\n")
        table.insert(tk.END, "-"*55 + "\n")
        for player, answer, elapsed in self.answers_this_round:
            table.insert(tk.END, f"{player:<15}{answer:<30}{elapsed:.2f}\n")
        table.config(state=tk.DISABLED)
        table.pack()
        ttk.Button(self.master, text="Далее", command=lambda: self.next_round_or_end(chooser)).pack(pady=10)

    def next_round_or_end(self, chooser):
        if self.marathon_mode:
            if any(ans.lower() != self.current_answer.lower() for _, ans, _ in self.answers_this_round):
                self.end_game()
                return
            if len(self.used_questions) == len(self.questions):
                self.end_game()
                return
            self.show_topic_selection(chooser)
        elif self.round < self.max_rounds:
            self.round += 1
            self.show_topic_selection(chooser)
        else:
            self.end_game()

    def end_game(self):
        self.clear_widgets()
        if self.team_mode:
            winner_idx = 0 if self.team_scores[0] > self.team_scores[1] else 1
            result = f"Победила {self.team_names[winner_idx]}!\nСчёт:\n{self.team_names[0]}: {self.team_scores[0]}\n{self.team_names[1]}: {self.team_scores[1]}"
        elif self.marathon_mode:
            result = f"Марафон окончен!\nВы ответили на {len(self.used_questions)} вопросов подряд без ошибок."
        else:
            winner = max(self.scores, key=lambda p: self.scores[p])
            result = f"Победитель: {winner}!\nСчёт:\n" + "\n".join([f"{p}: {self.scores[p]}" for p in self.player_names])
        ttk.Label(self.master, text="Игра окончена").pack(pady=10)
        ttk.Label(self.master, text=result).pack(pady=10)
        self.new_game_button.pack(pady=10)

    def restart_game(self):
        self.player_names = []
        self.scores = {}
        self.current_player_index = 0
        self.round = 1
        self.used_questions = set()
        self.answers_this_round = []
        self.team_scores = [0, 0]
        self.marathon_mode = False
        self.show_mode_selection()

    def clear_widgets(self):
        for widget in self.master.winfo_children():
            widget.pack_forget()

    def load_questions_online(self):
        url = "https://gist.githubusercontent.com/egorovpavel/7e2e2e2e2e2e2e2e2e2e2e2e2e2e2e2e/raw/quiz_questions.json"
        try:
            response = requests.get(url, timeout=5)
            data = response.json()
            self.questions = [(q['topic'], q['level'], q['question'], q['answer']) for q in data]
            messagebox.showinfo("Успех", "Вопросы успешно загружены из интернета!")
        except Exception as e:
            messagebox.showerror("Ошибка", f"Не удалось загрузить вопросы: {e}")

    def start_marathon(self):
        self.marathon_mode = True
        self.max_rounds = 9999
        self.show_num_players_input()

root = tk.Tk()
quiz_app = QuizApp(root)
root.mainloop()
