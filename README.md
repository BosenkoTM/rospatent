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
# @54?>;0305BAO, GB> D09; parser.py A DC=:F859 process_file_to_jsonl =0E>48BAO @O4><
from parser import process_file_to_jsonl

# --- <?>@B 4;O Drag & Drop ---
from tkinterdnd2 import DND_FILES, TkinterDnD

# --- ;0AA->15@B:0 4;O 8=B53@0F88 ---
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

        # !>7405< 2A5 M;5<5=BK 8=B5@D59A0
        self.create_main_menu()
        self.create_settings_widgets()
        self.create_dnd_area()
        # ### : >AAB0=02;8205< ?@028;L=K9 <5B>4 A>740=8O :=>?>: ###
        self.create_action_buttons_area()
        self.create_log_widgets()

        # 5@2>=0G0;L=0O =0AB@>9:0
        self.toggle_augmentation_slider()
        self.reset_to_initial_state()
        self.log("@8;>65=85 3>B>2>. 5@5=5A8B5 D09; 2 2K45;5==CN >1;0ABL 8;8 2>A?>;L7C9B5AL <5=N '$09;'.")

    def create_main_menu(self):
        menubar = Menu(self.root)
        self.root.config(menu=menubar)
        self.file_menu = Menu(menubar, tearoff=0)
        self.file_menu.add_command(label="B:@KBL...", command=self.select_file_callback, accelerator="Ctrl+O")
        self.file_menu.add_command(label="!>E@0=8BL :0:...", command=self.save_file_callback, state="disabled", accelerator="Ctrl+S")
        self.file_menu.add_separator()
        self.file_menu.add_command(label="KE>4", command=self.root.quit)
        menubar.add_cascade(label="$09;", menu=self.file_menu)

        self.service_menu = Menu(menubar, tearoff=0)
        self.service_menu.add_command(label="!35=5@8@>20BL 40B0A5B", command=self.start_generation_thread, state="disabled")
        self.service_menu.add_command(label="!:0G0BL H01;>= (.csv)", command=self.download_template)
        self.service_menu.add_separator()
        self.service_menu.add_command(label="G8AB8BL ;>3", command=self.clear_log)
        menubar.add_cascade(label="!5@28A", menu=self.service_menu)
        
        help_menu = Menu(menubar, tearoff=0)
        help_menu.add_command(label=">:C<5=B0F8O", command=self.show_help)
        help_menu.add_command(label=" ?@>3@0<<5", command=self.show_about)
        menubar.add_cascade(label="><>IL", menu=help_menu)
        
        self.root.bind("<Control-o>", lambda event: self.select_file_callback())
        self.root.bind("<Control-s>", lambda event: self.save_file_callback())

    def create_settings_widgets(self):
        parent = customtkinter.CTkFrame(self.root, height=150)
        parent.grid(row=0, column=0, padx=20, pady=10, sticky="ew")
        parent.grid_columnconfigure((0, 1), weight=1)
        class_frame = customtkinter.CTkFrame(parent, fg_color="transparent")
        class_frame.grid(row=0, column=0, padx=10, pady=10, sticky="ew")
        class_label = customtkinter.CTkLabel(class_frame, text="&5;52>9 C@>25=L:", font=customtkinter.CTkFont(weight="bold"))
        class_label.pack(anchor="w")
        self.complexity_combobox = customtkinter.CTkComboBox(class_frame, values=["2B><0B8G5A:8", "Beginner", "Intermediate", "Advanced", "Expert"])
        self.complexity_combobox.pack(fill="x", expand=True)
        self.complexity_combobox.set("2B><0B8G5A:8")
        
        aug_frame = customtkinter.CTkFrame(parent, fg_color="transparent")
        aug_frame.grid(row=0, column=1, padx=10, pady=10, sticky="ew")
        self.aug_checkbox = customtkinter.CTkCheckBox(aug_frame, text=":;NG8BL 0C3<5=B0F8N", command=self.toggle_augmentation_slider, font=customtkinter.CTkFont(weight="bold"))
        self.aug_checkbox.pack(anchor="w")
        self.aug_slider = customtkinter.CTkSlider(aug_frame, from_=1, to=5, number_of_steps=4, command=self.update_slider_label)
        self.aug_slider.pack(fill="x", expand=True, pady=5)
        self.aug_slider_label = customtkinter.CTkLabel(aug_frame, text=">MDD8F85=B C25;8G5=8O: 1x")
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
        
        self.dnd_label = customtkinter.CTkLabel(self.dnd_frame, text="5@5=5A8B5 2 2K45;5==CN >1;0ABL D09; (.xlsx 8;8 .csv)\n\n", font=customtkinter.CTkFont(size=20))
        self.dnd_label.grid(row=0, column=0, sticky="nsew", padx=20, pady=20)
        
        filename_container = customtkinter.CTkFrame(self.dnd_frame, fg_color="transparent")
        filename_container.grid(row=1, column=0, pady=(0, 10), padx=20)
        self.dnd_filename_label = customtkinter.CTkLabel(filename_container, text="", font=customtkinter.CTkFont(size=14, slant="italic"))
        self.dnd_filename_label.grid(row=0, column=0)
        self.reset_file_button = customtkinter.CTkButton(filename_container, text="L", width=28, height=28, command=self.reset_to_initial_state)
        self.reset_file_button.grid(row=0, column=1, padx=(10, 0))
        
        dnd_button_container = customtkinter.CTkFrame(self.dnd_frame, fg_color="transparent")
        dnd_button_container.grid(row=2, column=0, pady=(0, 20), padx=20)
        self.dnd_button = customtkinter.CTkButton(dnd_button_container, text=";8 2K15@8B5 D09; 2@CG=CN", command=self.select_file_callback)
        self.dnd_button.grid(row=0, column=0)

    # ### : >AAB0=>2;5= ?@028;L=K9 <5B>4 A>740=8O :=>?>: ###
    def create_action_buttons_area(self):
        # $@59<, 2 :>B>@>< 1C4CB <5=OBLAO :=>?:8
        self.action_frame = customtkinter.CTkFrame(self.root, fg_color="transparent")
        self.action_frame.grid(row=2, column=0, padx=20, pady=5, sticky="ew")
        # 0AB@08205< 425 :>;>=:8 @02=>9 H8@8=K 4;O <0;5=L:8E :=>?>:
        self.action_frame.grid_columnconfigure(0, weight=1)
        self.action_frame.grid_columnconfigure(1, weight=1)
        
        # 1. >;LH0O :=>?:0 "!35=5@8@>20BL"
        self.generate_button = customtkinter.CTkButton(self.action_frame, text="!35=5@8@>20BL 40B0A5B", height=40, font=customtkinter.CTkFont(size=16, weight="bold"), command=self.start_generation_thread)
        
        # 2. 0;5=L:85 :=>?:8, :>B>@K5 ?>O2OBAO ?>A;5
        self.download_result_button = customtkinter.CTkButton(self.action_frame, text=" !:0G0BL @57C;LB0B", height=40, command=self.save_file_callback)
        self.restart_button = customtkinter.CTkButton(self.action_frame, text="= 03@C78BL =>2K9 D09;", height=40, command=self.reset_to_initial_state)

    def create_log_widgets(self):
        parent = customtkinter.CTkFrame(self.root)
        parent.grid(row=3, column=0, padx=20, pady=10, sticky="nsew")
        parent.grid_rowconfigure(1, weight=1)
        parent.grid_columnconfigure(0, weight=1)
        log_header = customtkinter.CTkFrame(parent, fg_color="transparent")
        log_header.grid(row=0, column=0, padx=20, pady=(10, 0), sticky="ew")
        log_header.grid_columnconfigure(0, weight=1)
        log_label = customtkinter.CTkLabel(log_header, text=">3 2K?>;=5=8O", font=customtkinter.CTkFont(weight="bold"))
        log_label.grid(row=0, column=0, sticky="w")
        self.clear_log_button = customtkinter.CTkButton(log_header, text="G8AB8BL", width=80, command=self.clear_log)
        self.clear_log_button.grid(row=0, column=1, sticky="e")
        self.log_textbox = customtkinter.CTkTextbox(parent)
        self.log_textbox.grid(row=1, column=0, padx=20, pady=10, sticky="nsew")
        self.progressbar = customtkinter.CTkProgressBar(parent)
        self.progressbar.grid(row=2, column=0, padx=20, pady=(0, 10), sticky="ew")
        self.progressbar.set(0)

    # ### : >38:0 C?@02;5=8O :=>?:0<8 ###
    def reset_to_initial_state(self):
        self.input_filepath = ""
        self.dnd_frame.configure(fg_color=self.DND_AWAITING_BG, border_color=self.DND_AWAITING_BG)
        self.dnd_filename_label.configure(text="$09; =5 2K1@0=")
        self.reset_file_button.grid_remove()
        
        self.download_result_button.grid_remove()
        self.restart_button.grid_remove()
        self.generate_button.grid(row=0, column=0, columnspan=2, sticky="ew")
        
        self.update_generate_button_state(ready=False)
        self.file_menu.entryconfigure("!>E@0=8BL :0:...", state="disabled")
        
        if hasattr(self, 'log_textbox'):
            self.log("=B5@D59A A1@>H5=. 6840=85 =>2>3> D09;0.")

    def update_generate_button_state(self, ready: bool):
        if ready:
            self.generate_button.configure(state="normal", fg_color=self.BUTTON_ENABLED_COLOR)
            self.service_menu.entryconfigure("!35=5@8@>20BL 40B0A5B", state="normal")
        else:
            self.generate_button.configure(state="disabled", fg_color=self.BUTTON_DISABLED_COLOR)
            self.service_menu.entryconfigure("!35=5@8@>20BL 40B0A5B", state="disabled")
    
    def process_selected_file(self, filepath):
        # @8 2K1>@5 =>2>3> D09;0 2A5340 2>72@0I05<AO : 1>;LH>9 :=>?:5 "!35=5@8@>20BL"
        self.reset_to_initial_state()
        
        self.input_filepath = filepath
        filename = os.path.basename(filepath)
        self.dnd_filename_label.configure(text=f"K1@0= D09;: {filename}")
        self.log(f"K1@0= D09;: {filename}")
        self.update_generate_button_state(ready=True)
        self.dnd_frame.configure(fg_color=self.DND_READY_BG, border_color=self.DND_READY_BG)
        self.reset_file_button.grid()
        self.file_menu.entryconfigure("!>E@0=8BL :0:...", state="disabled")
        
    def start_generation_thread(self):
        if not self.input_filepath:
            messagebox.showerror("H81:0", "!=0G0;0 2K15@8B5 8;8 ?5@5B0I8B5 D09;!")
            return
        
        # ;>:8@C5< 2A5 :=>?:8 =0 2@5<O 35=5@0F88
        self.generate_button.configure(state="disabled")
        self.download_result_button.configure(state="disabled")
        self.restart_button.configure(state="disabled")

        self.progressbar.set(0)
        self.progressbar.start()
        self.log("="*40)
        self.log("0G8=0N 35=5@0F8N 40B0A5B0...")
        thread = threading.Thread(target=self.run_generation, daemon=True)
        thread.start()
        
    def generation_finished(self, initial_count, total_count):
        self.log("5=5@0F8O CA?5H=> 7025@H5=0!")
        if initial_count != total_count and total_count > 0:
            self.log(f"0945=> 8 >BD8;LB@>20=> 70?8A59: {initial_count}")
        self.log(f"B>3>2>5 :>;-2> 70?8A59: {total_count}")
        if total_count == 0:
            self.log(":  57C;LB0B ?CAB>9. $09; =5 1C45B A>45@60BL 40==KE.")
        
        self.progressbar.stop()
        self.progressbar.set(1)
        
        # ### : 5=O5< :=>?:8 ###
        self.generate_button.grid_remove()
        self.download_result_button.grid(row=0, column=0, padx=(0, 5), sticky="ew")
        self.restart_button.grid(row=0, column=1, padx=(5, 0), sticky="ew")
        
        # :B828@C5< :=>?:8 ?>A;5 35=5@0F88
        self.download_result_button.configure(state="normal")
        self.restart_button.configure(state="normal")
        self.file_menu.entryconfigure("!>E@0=8BL :0:...", state="normal")

    def generation_failed(self, error):
        self.log(f"(: {error}")
        messagebox.showerror("H81:0 35=5@0F88", str(error))
        self.progressbar.stop()
        self.progressbar.set(0)
        # >72@0I05< UI 2 A>AB>O=85 "3>B>2 : ?>2B>@=>9 ?>?KB:5"
        self.reset_to_initial_state()
        # > 5A;8 D09; 1K; 2K1@0=, A=>20 0:B828@C5< :=>?:C "!35=5@8@>20BL"
        if self.input_filepath:
            self.process_selected_file(self.input_filepath)

    # ### : #;CGH5==0O DC=:F8O A>E@0=5=8O A ?@>25@:0<8 ###
    def save_file_callback(self):
        if self.file_menu.entrycget("!>E@0=8BL :0:...", "state") == "disabled":
            messagebox.showwarning("5B 40==KE", "!=0G0;0 CA?5H=> A35=5@8@C9B5 D09; 4;O A>E@0=5=8O!")
            return

        if not os.path.exists(self.output_filename):
            messagebox.showerror("H81:0 A>E@0=5=8O", f"5 C40;>AL =09B8 ?@><56CB>G=K9 D09; '{self.output_filename}'. >?@>1C9B5 A35=5@8@>20BL 40==K5 A=>20.")
            return

        if os.path.getsize(self.output_filename) == 0:
            messagebox.showwarning(" 57C;LB0B ?CAB", "!35=5@8@>20==K9 D09; =5 A>45@68B 40==KE (?CAB).\n\n!>E@0=5=85 >B<5=5=>. @>25@LB5 =0AB@>9:8 D8;LB@0F88 8 8AE>4=K9 D09;.")
            return

        save_path = filedialog.asksaveasfilename(defaultextension=".jsonl", initialfile=self.output_filename, title="!>E@0=8BL @57C;LB0B :0:...", filetypes=[("JSON Lines", "*.jsonl")])
        if save_path:
            try:
                shutil.copy(self.output_filename, save_path)
                self.log(f"$09; CA?5H=> A>E@0=5= 2: {save_path}")
                messagebox.showinfo("#A?5E", f"$09; A>E@0=5= 2:\n{save_path}")
            except Exception as e:
                self.log(f"H81:0 A>E@0=5=8O: {e}")
                messagebox.showerror("H81:0", f"5 C40;>AL A>E@0=8BL D09;:\n{e}")

    def run_generation(self):
        try:
            level = self.complexity_combobox.get()
            is_augment_enabled = self.aug_checkbox.get() == 1
            factor = int(self.aug_slider.get())
            self.log(f"0AB@>9:8: #@>25=L='{level}', C3<5=B0F8O={is_augment_enabled}, >MDD.={factor}x")
            initial_count, total_count = process_file_to_jsonl(self.input_filepath, self.output_filename, level, is_augment_enabled, factor)
            self.root.after(100, lambda: self.generation_finished(initial_count, total_count))
        except Exception as e:
            self.root.after(100, lambda err=e: self.generation_failed(err))
    
    # --- AB0;L=K5 E5;?5@K 157 87<5=5=89 ---
    def download_template(self):
        headers = ['id', 'domain', 'domain_description', 'sql_complexity', 'sql_complexity_description', 'sql_task_type', 'sql_task_type_description', 'sql_prompt', 'sql_context', 'sql', 'sql_explanation', 'prompt_variation_1', 'sql_variation_1', 'prompt_variation_2', 'sql_variation_2']
        save_path = filedialog.asksaveasfilename(defaultextension=".csv", initialfile="template.csv", title="!>E@0=8BL H01;>= :0:...", filetypes=[("CSV (@0745;8B5;8 - 70?OBK5)", "*.csv")])
        if not save_path: return
        try:
            with open(save_path, 'w', newline='', encoding='utf-8') as f:
                csv.writer(f).writerow(headers)
            self.log(f"(01;>= CA?5H=> A>E@0=5= 2: {save_path}")
            messagebox.showinfo("#A?5E", f"(01;>= 'template.csv' CA?5H=> A>E@0=5=!")
        except Exception as e:
            self.log(f"( ?@8 A>E@0=5=88 H01;>=0: {e}")
            messagebox.showerror("H81:0", f"5 C40;>AL A>E@0=8BL D09; H01;>=0:\n{e}")

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
            messagebox.showwarning("525@=K9 D09;", f">6=> ?5@5B0A:820BL B>;L:> D09;K .xlsx 8 .csv.\nK ?5@5B0I8;8: {filepath}")

    def select_file_callback(self):
        filepath = filedialog.askopenfilename(title="K15@8B5 D09;", filetypes=(("Excel", "*.xlsx"), ("CSV", "*.csv"), ("A5 D09;K", "*.*")))
        if filepath: self.process_selected_file(filepath)
            
    def toggle_augmentation_slider(self):
        state = "normal" if self.aug_checkbox.get() else "disabled"
        self.aug_slider.configure(state=state)
        self.aug_slider_label.configure(state=state)
    
    def update_slider_label(self, value):
        self.aug_slider_label.configure(text=f">MDD8F85=B C25;8G5=8O: {int(value)}x")

    def clear_log(self):
        self.log_textbox.configure(state="normal")
        self.log_textbox.delete("1.0", "end")
        self.log_textbox.configure(state="disabled")
        self.log("AB>@8O >G8I5=0.")

    def log(self, message):
        self.log_textbox.configure(state="normal")
        self.log_textbox.insert("end", f"{message}\n")
        self.log_textbox.configure(state="disabled")
        self.log_textbox.see("end")
        
    def show_help(self): webbrowser.open_new_tab("https://github.com")
    def show_about(self): messagebox.showinfo(" ?@>3@0<<5", "SQL Dataset Generator LLM\n\n5@A8O 2.0\n 07@01>B0=> 2 @0<:0E ?@>5:B0  >A?0B5=B0.")

if __name__ == "__main__":
    customtkinter.set_appearance_mode("System")
    customtkinter.set_default_color_theme("blue")
    root = CTk_dnd()
    app = App(root)
    root.mainloop() 
