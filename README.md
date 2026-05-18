# stunning-garbanzo
BookTracker/
├── book_tracker.py      # Главный файл приложения
├── books.json           # Файл для хранения данных (создаётся автоматически)
├── README.md            # Документация
└── .gitignore           # Исключения для Git
import tkinter as tk
from tkinter import ttk, messagebox
import json
import os

class BookTracker:
    def init(self, root):
        self.root = root
        self.root.title("Book Tracker - Трекер прочитанных книг")
        self.root.geometry("850x600")
        self.root.resizable(True, True)

        self.books = []
        self.filtered_books = []
        self.data_file = "books.json"

        self.load_data()
        self.create_widgets()
        self.refresh_table()

    def create_widgets(self):
        # --- Рамка для ввода данных ---
        input_frame = tk.LabelFrame(self.root, text="Добавление новой книги", padx=10, pady=10)
        input_frame.pack(fill="x", padx=10, pady=5)

        # Поля ввода
        tk.Label(input_frame, text="Название:").grid(row=0, column=0, sticky="e", padx=5, pady=5)
        self.title_entry = tk.Entry(input_frame, width=30)
        self.title_entry.grid(row=0, column=1, padx=5, pady=5)

        tk.Label(input_frame, text="Автор:").grid(row=0, column=2, sticky="e", padx=5, pady=5)
        self.author_entry = tk.Entry(input_frame, width=30)
        self.author_entry.grid(row=0, column=3, padx=5, pady=5)

        tk.Label(input_frame, text="Жанр:").grid(row=1, column=0, sticky="e", padx=5, pady=5)
        self.genre_entry = tk.Entry(input_frame, width=30)
        self.genre_entry.grid(row=1, column=1, padx=5, pady=5)

        tk.Label(input_frame, text="Кол-во страниц:").grid(row=1, column=2, sticky="e", padx=5, pady=5)
        self.pages_entry = tk.Entry(input_frame, width=30)
        self.pages_entry.grid(row=1, column=3, padx=5, pady=5)

        # Кнопка добавления
        self.add_btn = tk.Button(input_frame, text="Добавить книгу", command=self.add_book, bg="lightgreen")
        self.add_btn.grid(row=2, column=0, columnspan=4, pady=10)

        # --- Рамка для фильтрации ---
        filter_frame = tk.LabelFrame(self.root, text="Фильтрация", padx=10, pady=10)
        filter_frame.pack(fill="x", padx=10, pady=5)

        tk.Label(filter_frame, text="Фильтр по жанру:").grid(row=0, column=0, padx=5)
        self.genre_filter_var = tk.StringVar()
        self.genre_filter_combo = ttk.Combobox(filter_frame, textvariable=self.genre_filter_var, width=20)
        self.genre_filter_combo.grid(row=0, column=1, padx=5)
        self.genre_filter_combo.bind("<<ComboboxSelected>>", lambda e: self.apply_filters())

        tk.Label(filter_frame, text="Страниц больше:").grid(row=0, column=2, padx=5)
        self.pages_filter_var = tk.StringVar()
        self.pages_filter_entry = tk.Entry(filter_frame, textvariable=self.pages_filter_var, width=10)
        self.pages_filter_entry.grid(row=0, column=3, padx=5)
        self.pages_filter_entry.bind("<KeyRelease>", lambda e: self.apply_filters())

        self.clear_filter_btn = tk.Button(filter_frame, text="Сбросить фильтры", command=self.clear_filters)
        self.clear_filter_btn.grid(row=0, column=4, padx=10)

        # --- Таблица для отображения книг ---
        table_frame = tk.Frame(self.root)
        table_frame.pack(fill="both", expand=True, padx=10, pady=5)

        # Создаём Treeview с прокруткой
        columns = ("Название", "Автор", "Жанр", "Страницы")
        self.tree = ttk.Treeview(table_frame, columns=columns, show="headings")

        for col in columns:
            self.tree.heading(col, text=col)
            self.tree.column(col, width=150)

        scrollbar = ttk.Scrollbar(table_frame, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)

        self.tree.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")

        # Кнопка удаления
        self.delete_btn = tk.Button(self.root, text="Удалить выбранную книгу", command=self.delete_book, bg="salmon")
        self.delete_btn.pack(pady=5)
        # Статус-бар
        self.status_var = tk.StringVar()
        self.status_var.set("Готово")
        status_bar = tk.Label(self.root, textvariable=self.status_var, bd=1, relief="sunken", anchor="w")
        status_bar.pack(fill="x", side="bottom")

    def add_book(self):
        title = self.title_entry.get().strip()
        author = self.author_entry.get().strip()
        genre = self.genre_entry.get().strip()
        pages_str = self.pages_entry.get().strip()

        # Валидация
        if not title or not author or not genre or not pages_str:
            messagebox.showerror("Ошибка", "Все поля должны быть заполнены!")
            return

        try:
            pages = int(pages_str)
            if pages <= 0:
                raise ValueError
        except ValueError:
            messagebox.showerror("Ошибка", "Количество страниц должно быть положительным числом!")
            return

        # Добавляем книгу
        book = {
            "title": title,
            "author": author,
            "genre": genre,
            "pages": pages
        }
        self.books.append(book)
        self.save_data()
        self.clear_input_fields()
        self.refresh_filters()
        self.apply_filters()
        self.status_var.set(f"Книга '{title}' добавлена")

    def delete_book(self):
        selected = self.tree.selection()
        if not selected:
            messagebox.showwarning("Внимание", "Выберите книгу для удаления!")
            return

        # Получаем индекс книги из текущего отфильтрованного списка
        for item in selected:
            item_values = self.tree.item(item, "values")
            # Ищем книгу по названию и автору
            for idx, book in enumerate(self.filtered_books):
                if book["title"] == item_values[0] and book["author"] == item_values[1]:
                    # Удаляем из основного списка
                    self.books.remove(book)
                    break
        self.save_data()
        self.refresh_filters()
        self.apply_filters()
        self.status_var.set("Книга удалена")

    def apply_filters(self):
        genre_filter = self.genre_filter_var.get().strip().lower()
        pages_filter = self.pages_filter_var.get().strip()

        self.filtered_books = self.books.copy()

        # Фильтр по жанру
        if genre_filter:
            self.filtered_books = [b for b in self.filtered_books if b["genre"].lower() == genre_filter]

        # Фильтр по страницам
        if pages_filter:
            try:
                pages_min = int(pages_filter)
                self.filtered_books = [b for b in self.filtered_books if b["pages"] > pages_min]
            except ValueError:
                pass

        self.refresh_table()

    def clear_filters(self):
        self.genre_filter_var.set("")
        self.pages_filter_var.set("")
        self.apply_filters()
        self.status_var.set("Фильтры сброшены")

    def refresh_table(self):
        # Очищаем таблицу
        for row in self.tree.get_children():
            self.tree.delete(row)

        # Заполняем отфильтрованными данными
        for book in self.filtered_books:
            self.tree.insert("", "end", values=(book["title"], book["author"], book["genre"], book["pages"]))

        self.status_var.set(f"Показано книг: {len(self.filtered_books)} / {len(self.books)}")

    def refresh_filters(self):
        # Обновляем список жанров в выпадающем списке
        genres = sorted(set(book["genre"] for book in self.books))
        self.genre_filter_combo["values"] = genres

    def clear_input_fields(self):
        self.title_entry.delete(0, tk.END)
        self.author_entry.delete(0, tk.END)
        self.genre_entry.delete(0, tk.END)
        self.pages_entry.delete(0, tk.END)

    def save_data(self):
        try:
            with open(self.data_file, "w", encoding="utf-8") as f:
                json.dump(self.books, f, ensure_ascii=False, indent=4)
        except Exception as e:
            messagebox.showerror("Ошибка", f"Не удалось сохранить данные: {e}")
