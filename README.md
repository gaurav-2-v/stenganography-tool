
import tkinter as tk
from tkinter import ttk, filedialog, messagebox
from PIL import Image, ImageTk
import stepic  # type: ignore
from Crypto.Cipher import AES  # type: ignore
from Crypto.Util.Padding import pad, unpad  # type: ignore
from Crypto.Random import get_random_bytes  # type: ignore
import base64
import hashlib
import os

class SteganographyApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Steganography Tool")
        self.root.geometry("1000x700")
        self.root.minsize(900, 600)

        # Variables
        self.image_path = tk.StringVar()
        
        # FIXED: Separated password variables
        self.embed_password = tk.StringVar()
        self.extract_password = tk.StringVar()
        
        self.encrypt_var = tk.BooleanVar()
        self.decrypt_var = tk.BooleanVar()

        # Setup UI
        self.setup_ui()
        self.set_theme()

    def setup_ui(self):
        top_frame = ttk.Frame(self.root)
        top_frame.pack(fill=tk.X)

        self.notebook = ttk.Notebook(self.root)
        self.notebook.pack(fill=tk.BOTH, expand=True)

        # Embed Tab
        self.embed_tab = ttk.Frame(self.notebook)
        self.notebook.add(self.embed_tab, text="Embed Message")
        self.embed_tab.columnconfigure(1, weight=1)
        self.embed_tab.rowconfigure(1, weight=1)

        # Extract Tab
        self.extract_tab = ttk.Frame(self.notebook)
        self.notebook.add(self.extract_tab, text="Extract Message")
        self.extract_tab.columnconfigure(1, weight=1)
        self.extract_tab.rowconfigure(1, weight=1)

        self.setup_embed_tab()
        self.setup_extract_tab()

        self.status_var = tk.StringVar(value="Ready")
        self.status_bar = ttk.Label(self.root, textvariable=self.status_var, relief=tk.SUNKEN)
        self.status_bar.pack(fill=tk.X, side=tk.BOTTOM)

    def setup_embed_tab(self):
        ttk.Label(self.embed_tab, text="Cover Image:").grid(row=0, column=0, sticky="w", padx=5, pady=5)
        ttk.Entry(self.embed_tab, textvariable=self.image_path, width=50).grid(row=0, column=1, sticky="ew", padx=5, pady=5)
        ttk.Button(self.embed_tab, text="Browse", command=self.browse_image).grid(row=0, column=2, padx=5, pady=5)

        self.embed_canvas = tk.Canvas(self.embed_tab, bg="#eee", highlightthickness=0)
        self.embed_canvas.grid(row=1, column=0, columnspan=3, sticky="nsew", padx=5, pady=5)
        self.embed_canvas.bind("<Configure>", lambda e: self.display_image(self.image_path.get(), self.embed_image_label, self.embed_canvas))
        self.embed_image_label = ttk.Label(self.embed_canvas)
        self.embed_canvas.create_window((0,0), window=self.embed_image_label, anchor="nw")

        ttk.Label(self.embed_tab, text="Secret Message:").grid(row=2, column=0, sticky="nw", padx=5, pady=5)
        self.message_text = tk.Text(self.embed_tab, height=10)
        self.message_text.grid(row=2, column=1, columnspan=2, sticky="nsew", padx=5, pady=5)

        ttk.Checkbutton(self.embed_tab, text="Encrypt Message", variable=self.encrypt_var,
                        command=self.toggle_encrypt_fields).grid(row=3, column=0, sticky="w", padx=5, pady=5)
        
        self.password_frame = ttk.Frame(self.embed_tab)
        self.password_frame.grid(row=4, column=0, columnspan=3, sticky="w", padx=5, pady=5)
        ttk.Label(self.password_frame, text="Password:").grid(row=0, column=0, padx=5, pady=5)
        
        # Link to embed_password
        ttk.Entry(self.password_frame, textvariable=self.embed_password, show="*", width=30).grid(row=0, column=1, padx=5, pady=5)
        self.password_frame.grid_remove()

        ttk.Button(self.embed_tab, text="Embed Message", command=self.embed_message).grid(row=5, column=0, columnspan=3, pady=10)

    def setup_extract_tab(self):
        ttk.Label(self.extract_tab, text="Stego Image:").grid(row=0, column=0, sticky="w", padx=5, pady=5)
        ttk.Entry(self.extract_tab, textvariable=self.image_path, width=50).grid(row=0, column=1, sticky="ew", padx=5, pady=5)
        ttk.Button(self.extract_tab, text="Browse", command=self.browse_extract_image).grid(row=0, column=2, padx=5, pady=5)

        self.extract_canvas = tk.Canvas(self.extract_tab, bg="#eee", highlightthickness=0)
        self.extract_canvas.grid(row=1, column=0, columnspan=3, sticky="nsew", padx=5, pady=5)
        self.extract_canvas.bind("<Configure>", lambda e: self.display_image(self.image_path.get(), self.extract_image_label, self.extract_canvas))
        self.extract_image_label = ttk.Label(self.extract_canvas)
        self.extract_canvas.create_window((0,0), window=self.extract_image_label, anchor="nw")

        ttk.Checkbutton(self.extract_tab, text="Decrypt Message", variable=self.decrypt_var,
                        command=self.toggle_decrypt_fields).grid(row=2, column=0, sticky="w", padx=5, pady=5)
        
        self.extract_password_frame = ttk.Frame(self.extract_tab)
        self.extract_password_frame.grid(row=3, column=0, columnspan=3, sticky="w", padx=5, pady=5)
        ttk.Label(self.extract_password_frame, text="Password:").grid(row=0, column=0, padx=5, pady=5)
        
        # Link to extract_password
        ttk.Entry(self.extract_password_frame, textvariable=self.extract_password, show="*", width=30).grid(row=0, column=1, padx=5, pady=5)
        self.extract_password_frame.grid_remove()

        ttk.Button(self.extract_tab, text="Extract Message", command=self.extract_message).grid(row=4, column=0, columnspan=3, pady=10)
        ttk.Label(self.extract_tab, text="Extracted Message:").grid(row=5, column=0, sticky="nw", padx=5, pady=5)
        self.extracted_message_text = tk.Text(self.extract_tab, height=10, state=tk.DISABLED)
        self.extracted_message_text.grid(row=5, column=1, columnspan=2, sticky="nsew", padx=5, pady=5)

    def encrypt_message(self, message, password):
        try:
            salt = get_random_bytes(16)
            key = hashlib.pbkdf2_hmac('sha256', password.encode(), salt, 100000, dklen=32)
            cipher = AES.new(key, AES.MODE_CBC)
            ct_bytes = cipher.encrypt(pad(message.encode(), AES.block_size))
            encrypted_data = salt + cipher.iv + ct_bytes
            return base64.b64encode(encrypted_data).decode('utf-8')
        except Exception as e:
            messagebox.showerror("Encryption Error", str(e))
            return None

    def decrypt_message(self, encrypted_message, password):
        try:
            encrypted_data = base64.b64decode(encrypted_message)
            salt = encrypted_data[:16]
            iv = encrypted_data[16:32]
            ct = encrypted_data[32:]
            key = hashlib.pbkdf2_hmac('sha256', password.encode(), salt, 100000, dklen=32)
            cipher = AES.new(key, AES.MODE_CBC, iv)
            pt = unpad(cipher.decrypt(ct), AES.block_size)
            return pt.decode('utf-8')
        except Exception as e:
            messagebox.showerror("Decryption Error", "Incorrect password or corrupted data.")
            return None

    def browse_image(self):
        file_path = filedialog.askopenfilename(filetypes=[("Image Files", "*.png;*.jpg;*.jpeg;*.bmp")])
        if file_path:
            self.image_path.set(file_path)
            self.display_image(file_path, self.embed_image_label, self.embed_canvas)

    def browse_extract_image(self):
        file_path = filedialog.askopenfilename(filetypes=[("Image Files", "*.png;*.jpg;*.jpeg;*.bmp")])
        if file_path:
            self.image_path.set(file_path)
            self.display_image(file_path, self.extract_image_label, self.extract_canvas)

    def display_image(self, path, label_widget, canvas_widget):
        if not path or not os.path.exists(path):
            return
        try:
            image = Image.open(path)
            canvas_width = canvas_widget.winfo_width() or 500
            canvas_height = canvas_widget.winfo_height() or 500
            img_ratio = image.width / image.height
            canvas_ratio = canvas_width / canvas_height
            if img_ratio > canvas_ratio:
                new_width = canvas_width
                new_height = int(canvas_width / img_ratio)
            else:
                new_height = canvas_height
                new_width = int(canvas_height * img_ratio)
            image = image.resize((new_width, new_height), Image.Resampling.LANCZOS)
            photo = ImageTk.PhotoImage(image)
            label_widget.configure(image=photo)
            label_widget.image = photo
            canvas_widget.configure(scrollregion=(0,0,new_width,new_height))
        except Exception as e:
            messagebox.showerror("Error", str(e))

    def embed_message(self):
        image_path = self.image_path.get()
        message = self.message_text.get("1.0", tk.END).strip()
        if not image_path or not message:
            messagebox.showerror("Error", "Please select image and enter message")
            return
        
        if self.encrypt_var.get():
            password = self.embed_password.get() # Corrected variable
            if not password:
                messagebox.showerror("Error", "Enter encryption password")
                return
            encrypted_payload = self.encrypt_message(message, password)
            if encrypted_payload:
                message = f"ENCRYPTED:{encrypted_payload}"
            else:
                return

        try:
            image = Image.open(image_path)
            encoded_image = stepic.encode(image, message.encode('utf-8'))
            save_path = filedialog.asksaveasfilename(defaultextension=".png",
                                                     filetypes=[("PNG Files", "*.png"), ("All Files", "*.*")])
            if save_path:
                encoded_image.save(save_path)
                messagebox.showinfo("Success", f"Message embedded successfully!\nSaved to: {save_path}")
                self.status_var.set("Message embedded successfully")
        except Exception as e:
            messagebox.showerror("Error", str(e))
            self.status_var.set("Error embedding message")

    def extract_message(self):
        image_path = self.image_path.get()
        if not image_path:
            messagebox.showerror("Error", "Please select image")
            return
        try:
            image = Image.open(image_path)
            extracted_data = stepic.decode(image)
            
            # Stepic returns bytes, convert to string
            if isinstance(extracted_data, bytes):
                extracted_data = extracted_data.decode('utf-8')

            if extracted_data.startswith("ENCRYPTED:"):
                if not self.decrypt_var.get():
                    messagebox.showwarning("Warning", "Encrypted message detected! Please check 'Decrypt Message'.")
                    self.show_extracted_message(extracted_data)
                    return
                
                password = self.extract_password.get() # Corrected variable
                if not password:
                    messagebox.showerror("Error", "Enter decryption password")
                    return
                
                encrypted_part = extracted_data.split("ENCRYPTED:")[1]
                decrypted = self.decrypt_message(encrypted_part, password)
                if decrypted:
                    self.show_extracted_message(decrypted)
            else:
                self.show_extracted_message(extracted_data)
        except Exception as e:
            messagebox.showerror("Error", "No message found or error in decoding.")

    def show_extracted_message(self, text):
        self.extracted_message_text.config(state=tk.NORMAL)
        self.extracted_message_text.delete("1.0", tk.END)
        self.extracted_message_text.insert(tk.END, text)
        self.extracted_message_text.config(state=tk.DISABLED)
        self.status_var.set("Message extracted successfully")

    def toggle_encrypt_fields(self):
        self.password_frame.grid() if self.encrypt_var.get() else self.password_frame.grid_remove()
    
    def toggle_decrypt_fields(self):
        self.extract_password_frame.grid() if self.decrypt_var.get() else self.extract_password_frame.grid_remove()
    
    def set_theme(self):
        style = ttk.Style()
        style.theme_use('clam')
        style.configure(".", background="#f0f0f0", foreground="#202020", fieldbackground="#ffffff")
        style.configure("TEntry", fieldbackground="#ffffff")
        style.configure("TCheckbutton", background="#f0f0f0", foreground="#202020")
        style.configure("TLabel", background="#f0f0f0", foreground="#202020")
        style.configure("TButton", background="#e0e0e0", foreground="#202020")
        style.configure("TNotebook", background="#f0f0f0", borderwidth=0)
        style.configure("TNotebook.Tab", background="#e0e0e0", foreground="#202020")
        style.map("TNotebook.Tab", background=[("selected", "#f0f0f0")])
        self.root.configure(bg="#f0f0f0")
        self.embed_canvas.configure(bg="#eee", highlightthickness=0)
        self.extract_canvas.configure(bg="#eee", highlightthickness=0)
        self.status_bar.configure(background="#f0f0f0", foreground="#202020")
        self.message_text.configure(bg="#ffffff", fg="#202020", insertbackground="#202020")
        self.extracted_message_text.configure(bg="#ffffff", fg="#202020", insertbackground="#202020")

if __name__ == "__main__":
    root = tk.Tk()
    app = SteganographyApp(root)
    root.mainloop()
