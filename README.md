# app.py
import tkinter
from tkinter import Menu
import customtkinter
from tkinter import filedialog, messagebox
import threading
import os
import webbrowser
import csv
import shutil
# Предполагается, что файл parser.py с функцией process_file_to_jsonl находится рядом
from parser import process_file_to_jsonl

# --- Импорт для Drag & Drop ---
from tkinterdnd2 import DND_FILES, TkinterDnD

# --- Класс-обертка для интеграции ---
class CTk_dnd(customtkinter.CTk, TkinterDnD.DnDWrapper):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.TkdndVersion = TkinterDnD._require(self)


class App:
    def __init__(self, root: CTk_dnd):
        self.root = root
        self.root.title("SQL Dataset Generator LLM")
        self.root.geometry("800x750")
        
        self.DND_READY_BG = ("#E0FFE0", "#004225")
        self.DND_AWAITING_BG = ("#FFD1D1", "#5C1F1F")
        self.BUTTON_DISABLED_COLOR = ("#C0C0C0", "#505050")
        self.BUTTON_ENABLED_COLOR = customtkinter.ThemeManager.theme["CTkButton"]["fg_color"]
        
        self.input_filepath = ""
        self.output_filename = "dataset.jsonl"
        
        self.root.grid_rowconfigure(1, weight=1)
        self.root.grid_rowconfigure(3, minsize=200)
        self.root.grid_columnconfigure(0, weight=1)

        # Создаем все элементы интерфейса
        self.create_main_menu()
        self.create_settings_widgets()
        self.create_dnd_area()
        # ### ИЗМЕНЕНИЕ: Восстанавливаем правильный метод создания кнопок ###
        self.create_action_buttons_area()
        self.create_log_widgets()

        # Первоначальная настройка
        self.toggle_augmentation_slider()
        self.reset_to_initial_state()
        self.log("Приложение готово. Перенесите файл в выделенную область или воспользуйтесь меню 'Файл'.")

    def create_main_menu(self):
        menubar = Menu(self.root)
        self.root.config(menu=menubar)
        self.file_menu = Menu(menubar, tearoff=0)
        self.file_menu.add_command(label="Открыть...", command=self.select_file_callback, accelerator="Ctrl+O")
        self.file_menu.add_command(label="Сохранить как...", command=self.save_file_callback, state="disabled", accelerator="Ctrl+S")
        self.file_menu.add_separator()
        self.file_menu.add_command(label="Выход", command=self.root.quit)
        menubar.add_cascade(label="Файл", menu=self.file_menu)

        self.service_menu = Menu(menubar, tearoff=0)
        self.service_menu.add_command(label="Сгенерировать датасет", command=self.start_generation_thread, state="disabled")
        self.service_menu.add_command(label="Скачать шаблон (.csv)", command=self.download_template)
        self.service_menu.add_separator()
        self.service_menu.add_command(label="Очистить лог", command=self.clear_log)
        menubar.add_cascade(label="Сервис", menu=self.service_menu)
        
        help_menu = Menu(menubar, tearoff=0)
        help_menu.add_command(label="Документация", command=self.show_help)
        help_menu.add_command(label="О программе", command=self.show_about)
        menubar.add_cascade(label="Помощь", menu=help_menu)
        
        self.root.bind("<Control-o>", lambda event: self.select_file_callback())
        self.root.bind("<Control-s>", lambda event: self.save_file_callback())

    def create_settings_widgets(self):
        parent = customtkinter.CTkFrame(self.root, height=150)
        parent.grid(row=0, column=0, padx=20, pady=10, sticky="ew")
        parent.grid_columnconfigure((0, 1), weight=1)
        class_frame = customtkinter.CTkFrame(parent, fg_color="transparent")
        class_frame.grid(row=0, column=0, padx=10, pady=10, sticky="ew")
        class_label = customtkinter.CTkLabel(class_frame, text="Целевой уровень:", font=customtkinter.CTkFont(weight="bold"))
        class_label.pack(anchor="w")
        self.complexity_combobox = customtkinter.CTkComboBox(class_frame, values=["Автоматически", "Beginner", "Intermediate", "Advanced", "Expert"])
        self.complexity_combobox.pack(fill="x", expand=True)
        self.complexity_combobox.set("Автоматически")
        
        aug_frame = customtkinter.CTkFrame(parent, fg_color="transparent")
        aug_frame.grid(row=0, column=1, padx=10, pady=10, sticky="ew")
        self.aug_checkbox = customtkinter.CTkCheckBox(aug_frame, text="Включить аугментацию", command=self.toggle_augmentation_slider, font=customtkinter.CTkFont(weight="bold"))
        self.aug_checkbox.pack(anchor="w")
        self.aug_slider = customtkinter.CTkSlider(aug_frame, from_=1, to=5, number_of_steps=4, command=self.update_slider_label)
        self.aug_slider.pack(fill="x", expand=True, pady=5)
        self.aug_slider_label = customtkinter.CTkLabel(aug_frame, text="Коэффициент увеличения: 1x")
        self.aug_slider_label.pack(anchor="e")
        self.aug_slider.set(1)
        
    def create_dnd_area(self):
        self.dnd_frame = customtkinter.CTkFrame(self.root, border_width=2)
        self.dnd_frame.grid(row=1, column=0, padx=20, pady=10, sticky="nsew")
        self.dnd_frame.grid_rowconfigure(0, weight=1)
        self.dnd_frame.grid_columnconfigure(0, weight=1)
        self.dnd_frame.drop_target_register(DND_FILES)
        self.dnd_frame.dnd_bind('<<Drop>>', self.handle_drop)
        self.dnd_frame.dnd_bind('<<DragEnter>>', self.on_drag_enter)
        self.dnd_frame.dnd_bind('<<DragLeave>>', self.on_drag_leave)
        self.dnd_frame.bind("<Enter>", self.on_mouse_enter)
        self.dnd_frame.bind("<Leave>", self.on_mouse_leave)
        
        self.dnd_label = customtkinter.CTkLabel(self.dnd_frame, text="Перенесите в выделенную область файл (.xlsx или .csv)\n\n", font=customtkinter.CTkFont(size=20))
        self.dnd_label.grid(row=0, column=0, sticky="nsew", padx=20, pady=20)
        
        filename_container = customtkinter.CTkFrame(self.dnd_frame, fg_color="transparent")
        filename_container.grid(row=1, column=0, pady=(0, 10), padx=20)
        self.dnd_filename_label = customtkinter.CTkLabel(filename_container, text="", font=customtkinter.CTkFont(size=14, slant="italic"))
        self.dnd_filename_label.grid(row=0, column=0)
        self.reset_file_button = customtkinter.CTkButton(filename_container, text="❌", width=28, height=28, command=self.reset_to_initial_state)
        self.reset_file_button.grid(row=0, column=1, padx=(10, 0))
        
        dnd_button_container = customtkinter.CTkFrame(self.dnd_frame, fg_color="transparent")
        dnd_button_container.grid(row=2, column=0, pady=(0, 20), padx=20)
        self.dnd_button = customtkinter.CTkButton(dnd_button_container, text="Или выберите файл вручную", command=self.select_file_callback)
        self.dnd_button.grid(row=0, column=0)

    # ### ИЗМЕНЕНИЕ: Восстановлен правильный метод создания кнопок ###
    def create_action_buttons_area(self):
        # Фрейм, в котором будут меняться кнопки
        self.action_frame = customtkinter.CTkFrame(self.root, fg_color="transparent")
        self.action_frame.grid(row=2, column=0, padx=20, pady=5, sticky="ew")
        # Настраиваем две колонки равной ширины для маленьких кнопок
        self.action_frame.grid_columnconfigure(0, weight=1)
        self.action_frame.grid_columnconfigure(1, weight=1)
        
        # 1. Большая кнопка "Сгенерировать"
        self.generate_button = customtkinter.CTkButton(self.action_frame, text="Сгенерировать датасет", height=40, font=customtkinter.CTkFont(size=16, weight="bold"), command=self.start_generation_thread)
        
        # 2. Маленькие кнопки, которые появятся после
        self.download_result_button = customtkinter.CTkButton(self.action_frame, text="⬇️ Скачать результат", height=40, command=self.save_file_callback)
        self.restart_button = customtkinter.CTkButton(self.action_frame, text="🔄 Загрузить новый файл", height=40, command=self.reset_to_initial_state)

    def create_log_widgets(self):
        parent = customtkinter.CTkFrame(self.root)
        parent.grid(row=3, column=0, padx=20, pady=10, sticky="nsew")
        parent.grid_rowconfigure(1, weight=1)
        parent.grid_columnconfigure(0, weight=1)
        log_header = customtkinter.CTkFrame(parent, fg_color="transparent")
        log_header.grid(row=0, column=0, padx=20, pady=(10, 0), sticky="ew")
        log_header.grid_columnconfigure(0, weight=1)
        log_label = customtkinter.CTkLabel(log_header, text="Лог выполнения", font=customtkinter.CTkFont(weight="bold"))
        log_label.grid(row=0, column=0, sticky="w")
        self.clear_log_button = customtkinter.CTkButton(log_header, text="Очистить", width=80, command=self.clear_log)
        self.clear_log_button.grid(row=0, column=1, sticky="e")
        self.log_textbox = customtkinter.CTkTextbox(parent)
        self.log_textbox.grid(row=1, column=0, padx=20, pady=10, sticky="nsew")
        self.progressbar = customtkinter.CTkProgressBar(parent)
        self.progressbar.grid(row=2, column=0, padx=20, pady=(0, 10), sticky="ew")
        self.progressbar.set(0)

    # ### ИЗМЕНЕНИЕ: Логика управления кнопками ###
    def reset_to_initial_state(self):
        self.input_filepath = ""
        self.dnd_frame.configure(fg_color=self.DND_AWAITING_BG, border_color=self.DND_AWAITING_BG)
        self.dnd_filename_label.configure(text="Файл не выбран")
        self.reset_file_button.grid_remove()
        
        self.download_result_button.grid_remove()
        self.restart_button.grid_remove()
        self.generate_button.grid(row=0, column=0, columnspan=2, sticky="ew")
        
        self.update_generate_button_state(ready=False)
        self.file_menu.entryconfigure("Сохранить как...", state="disabled")
        
        if hasattr(self, 'log_textbox'):
            self.log("Интерфейс сброшен. Ожидание нового файла.")

    def update_generate_button_state(self, ready: bool):
        if ready:
            self.generate_button.configure(state="normal", fg_color=self.BUTTON_ENABLED_COLOR)
            self.service_menu.entryconfigure("Сгенерировать датасет", state="normal")
        else:
            self.generate_button.configure(state="disabled", fg_color=self.BUTTON_DISABLED_COLOR)
            self.service_menu.entryconfigure("Сгенерировать датасет", state="disabled")
    
    def process_selected_file(self, filepath):
        # При выборе нового файла всегда возвращаемся к большой кнопке "Сгенерировать"
        self.reset_to_initial_state()
        
        self.input_filepath = filepath
        filename = os.path.basename(filepath)
        self.dnd_filename_label.configure(text=f"Выбран файл: {filename}")
        self.log(f"Выбран файл: {filename}")
        self.update_generate_button_state(ready=True)
        self.dnd_frame.configure(fg_color=self.DND_READY_BG, border_color=self.DND_READY_BG)
        self.reset_file_button.grid()
        self.file_menu.entryconfigure("Сохранить как...", state="disabled")
        
    def start_generation_thread(self):
        if not self.input_filepath:
            messagebox.showerror("Ошибка", "Сначала выберите или перетащите файл!")
            return
        
        # Блокируем все кнопки на время генерации
        self.generate_button.configure(state="disabled")
        self.download_result_button.configure(state="disabled")
        self.restart_button.configure(state="disabled")

        self.progressbar.set(0)
        self.progressbar.start()
        self.log("="*40)
        self.log("Начинаю генерацию датасета...")
        thread = threading.Thread(target=self.run_generation, daemon=True)
        thread.start()
        
    def generation_finished(self, initial_count, total_count):
        self.log("Генерация успешно завершена!")
        if initial_count != total_count and total_count > 0:
            self.log(f"Найдено и отфильтровано записей: {initial_count}")
        self.log(f"Итоговое кол-во записей: {total_count}")
        if total_count == 0:
            self.log("ВНИМАНИЕ: Результат пустой. Файл не будет содержать данных.")
        
        self.progressbar.stop()
        self.progressbar.set(1)
        
        # ### ИЗМЕНЕНИЕ: Меняем кнопки ###
        self.generate_button.grid_remove()
        self.download_result_button.grid(row=0, column=0, padx=(0, 5), sticky="ew")
        self.restart_button.grid(row=0, column=1, padx=(5, 0), sticky="ew")
        
        # Активируем кнопки после генерации
        self.download_result_button.configure(state="normal")
        self.restart_button.configure(state="normal")
        self.file_menu.entryconfigure("Сохранить как...", state="normal")

    def generation_failed(self, error):
        self.log(f"ОШИБКА: {error}")
        messagebox.showerror("Ошибка генерации", str(error))
        self.progressbar.stop()
        self.progressbar.set(0)
        # Возвращаем UI в состояние "готов к повторной попытке"
        self.reset_to_initial_state()
        # Но если файл был выбран, снова активируем кнопку "Сгенерировать"
        if self.input_filepath:
            self.process_selected_file(self.input_filepath)

    # ### ИЗМЕНЕНИЕ: Улучшенная функция сохранения с проверками ###
    def save_file_callback(self):
        if self.file_menu.entrycget("Сохранить как...", "state") == "disabled":
            messagebox.showwarning("Нет данных", "Сначала успешно сгенерируйте файл для сохранения!")
            return

        if not os.path.exists(self.output_filename):
            messagebox.showerror("Ошибка сохранения", f"Не удалось найти промежуточный файл '{self.output_filename}'. Попробуйте сгенерировать данные снова.")
            return

        if os.path.getsize(self.output_filename) == 0:
            messagebox.showwarning("Результат пуст", "Сгенерированный файл не содержит данных (пуст).\n\nСохранение отменено. Проверьте настройки фильтрации и исходный файл.")
            return

        save_path = filedialog.asksaveasfilename(defaultextension=".jsonl", initialfile=self.output_filename, title="Сохранить результат как...", filetypes=[("JSON Lines", "*.jsonl")])
        if save_path:
            try:
                shutil.copy(self.output_filename, save_path)
                self.log(f"Файл успешно сохранен в: {save_path}")
                messagebox.showinfo("Успех", f"Файл сохранен в:\n{save_path}")
            except Exception as e:
                self.log(f"Ошибка сохранения: {e}")
                messagebox.showerror("Ошибка", f"Не удалось сохранить файл:\n{e}")

    def run_generation(self):
        try:
            level = self.complexity_combobox.get()
            is_augment_enabled = self.aug_checkbox.get() == 1
            factor = int(self.aug_slider.get())
            self.log(f"Настройки: Уровень='{level}', Аугментация={is_augment_enabled}, Коэфф.={factor}x")
            initial_count, total_count = process_file_to_jsonl(self.input_filepath, self.output_filename, level, is_augment_enabled, factor)
            self.root.after(100, lambda: self.generation_finished(initial_count, total_count))
        except Exception as e:
            self.root.after(100, lambda err=e: self.generation_failed(err))
    
    # --- Остальные хелперы без изменений ---
    def download_template(self):
        headers = ['id', 'domain', 'domain_description', 'sql_complexity', 'sql_complexity_description', 'sql_task_type', 'sql_task_type_description', 'sql_prompt', 'sql_context', 'sql', 'sql_explanation', 'prompt_variation_1', 'sql_variation_1', 'prompt_variation_2', 'sql_variation_2']
        save_path = filedialog.asksaveasfilename(defaultextension=".csv", initialfile="template.csv", title="Сохранить шаблон как...", filetypes=[("CSV (разделители - запятые)", "*.csv")])
        if not save_path: return
        try:
            with open(save_path, 'w', newline='', encoding='utf-8') as f:
                csv.writer(f).writerow(headers)
            self.log(f"Шаблон успешно сохранен в: {save_path}")
            messagebox.showinfo("Успех", f"Шаблон 'template.csv' успешно сохранен!")
        except Exception as e:
            self.log(f"ОШИБКА при сохранении шаблона: {e}")
            messagebox.showerror("Ошибка", f"Не удалось сохранить файл шаблона:\n{e}")

    def on_mouse_enter(self, event):
        border_color = self.DND_READY_BG if self.input_filepath else customtkinter.ThemeManager.theme["CTkButton"]["fg_color"]
        self.dnd_frame.configure(border_color=border_color)

    def on_mouse_leave(self, event):
        border_color = self.DND_READY_BG if self.input_filepath else self.DND_AWAITING_BG
        self.dnd_frame.configure(border_color=border_color)

    def on_drag_enter(self, event):
        if not self.input_filepath: self.dnd_frame.configure(fg_color=self.DND_READY_BG)
        return event.action

    def on_drag_leave(self, event):
        if not self.input_filepath: self.dnd_frame.configure(fg_color=self.DND_AWAITING_BG)

    def handle_drop(self, event):
        filepath = event.data.strip('{}')
        if os.path.isfile(filepath) and (filepath.lower().endswith(".xlsx") or filepath.lower().endswith(".csv")):
            self.process_selected_file(filepath)
        else:
            messagebox.showwarning("Неверный файл", f"Можно перетаскивать только файлы .xlsx и .csv.\nВы перетащили: {filepath}")

    def select_file_callback(self):
        filepath = filedialog.askopenfilename(title="Выберите файл", filetypes=(("Excel", "*.xlsx"), ("CSV", "*.csv"), ("Все файлы", "*.*")))
        if filepath: self.process_selected_file(filepath)
            
    def toggle_augmentation_slider(self):
        state = "normal" if self.aug_checkbox.get() else "disabled"
        self.aug_slider.configure(state=state)
        self.aug_slider_label.configure(state=state)
    
    def update_slider_label(self, value):
        self.aug_slider_label.configure(text=f"Коэффициент увеличения: {int(value)}x")

    def clear_log(self):
        self.log_textbox.configure(state="normal")
        self.log_textbox.delete("1.0", "end")
        self.log_textbox.configure(state="disabled")
        self.log("История очищена.")

    def log(self, message):
        self.log_textbox.configure(state="normal")
        self.log_textbox.insert("end", f"{message}\n")
        self.log_textbox.configure(state="disabled")
        self.log_textbox.see("end")
        
    def show_help(self): webbrowser.open_new_tab("https://github.com")
    def show_about(self): messagebox.showinfo("О программе", "SQL Dataset Generator LLM\n\nВерсия 2.0\nРазработано в рамках проекта Роспатента.")

if __name__ == "__main__":
    customtkinter.set_appearance_mode("System")
    customtkinter.set_default_color_theme("blue")
    root = CTk_dnd()
    app = App(root)
    root.mainloop()
