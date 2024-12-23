# toko-sewa-mobillimport mysql.connector
from prettytable import PrettyTable
import pyfiglet
from colorama import Fore, Style, init

# Inisialisasi colorama
init(autoreset=True)

# Fungsi untuk menampilkan header keren
def print_cool_header():
    ascii_art = pyfiglet.figlet_format("SEWA MOBIL")
    print(Fore.RED + Style.BRIGHT + ascii_art)
    print(Fore.CYAN + "=" * 40)
    print(Fore.YELLOW + "       SISTEM MANAJEMEN SEWA MOBIL")
    print(Fore.CYAN + "=" * 40 + "\n")

# Koneksi ke database
connection = mysql.connector.connect(
    host="localhost",
    user="root",
    password="",
    database="project"
)

# Fungsi untuk menambah data mobil
def add_car(connection):
    car_name = input("Masukkan nama mobil: ")
    plate_number = input("Masukkan plat nomor: ")
    price_per_day = input("Masukkan harga sewa per hari: ")
    cursor = connection.cursor()
    query = "INSERT INTO cars (car_name, plate_number, price_per_day, status) VALUES (%s, %s, %s, 'available')"
    cursor.execute(query, (car_name, plate_number, price_per_day))
    connection.commit()
    print(f"{cursor.rowcount} mobil berhasil ditambahkan.")

# Fungsi untuk menampilkan data mobil
def show_cars(connection):
    cursor = connection.cursor()
    query = "SELECT * FROM cars"
    cursor.execute(query)
    result = cursor.fetchall()

    if cursor.rowcount <= 0:
        print("Tidak ada data mobil.")
    else:
        table = PrettyTable(["ID", "Nama Mobil", "Plat Nomor", "Harga/Hari", "Status"])
        for car in result:
            table.add_row(car)
        print(table)

# Fungsi untuk menambah data pelanggan
def add_customer(connection):
    name = input("Masukkan nama pelanggan: ")
    address = input("Masukkan alamat pelanggan: ")
    cursor = connection.cursor()
    query = "INSERT INTO customers (name, address) VALUES (%s, %s)"
    cursor.execute(query, (name, address))
    connection.commit()
    print(f"{cursor.rowcount} pelanggan berhasil ditambahkan.")

# Fungsi untuk menampilkan data pelanggan
def show_customers(connection):
    cursor = connection.cursor()
    query = """
        SELECT customers.customer_id, customers.name, customers.address, 
               cars.car_name, cars.plate_number, rentals.start_date, rentals.end_date
        FROM customers
        LEFT JOIN rentals ON customers.customer_id = rentals.customer_id
        LEFT JOIN cars ON rentals.car_id = cars.car_id
    """
    cursor.execute(query)
    result = cursor.fetchall()

    if cursor.rowcount <= 0:
        print("Tidak ada data pelanggan.")
    else:
        table = PrettyTable(["ID", "Nama Pelanggan", "Alamat", "Mobil Disewa", "Plat Nomor", "Mulai", "Selesai"])
        for data in result:
            mobil_disewa = data[3] if data[3] else "Tidak ada"
            table.add_row([data[0], data[1], data[2], mobil_disewa, data[4], data[5], data[6]])
        print(table)

# Fungsi untuk menyewa mobil
def rent_car(connection):
    show_cars(connection)
    car_id = input("Masukkan ID mobil yang akan disewa: ")
    show_customers(connection)
    customer_id = input("Masukkan ID pelanggan: ")
    start_date = input("Masukkan tanggal mulai (YYYY-MM-DD): ")
    end_date = input("Masukkan tanggal selesai (YYYY-MM-DD): ")

    cursor = connection.cursor()
    cursor.execute("SELECT price_per_day FROM cars WHERE car_id = %s AND status = 'available'", (car_id,))
    car = cursor.fetchone()
    if car:
        price_per_day = car[0]
        cursor.execute("SELECT DATEDIFF(%s, %s)", (end_date, start_date))
        total_days = cursor.fetchone()[0]
        total_price = total_days * price_per_day

        rent_query = "INSERT INTO rentals (customer_id, car_id, start_date, end_date, total_price) VALUES (%s, %s, %s, %s, %s)"
        cursor.execute(rent_query, (customer_id, car_id, start_date, end_date, total_price))

        update_car_query = "UPDATE cars SET status = 'rented' WHERE car_id = %s"
        cursor.execute(update_car_query, (car_id,))
        connection.commit()

        print(f"Mobil berhasil disewa. Total harga: {total_price}")
    else:
        print("Mobil tidak tersedia untuk disewa.")

# Fungsi untuk mengembalikan mobil
def return_car(connection):
    rental_id = input("Masukkan ID rental: ")
    cursor = connection.cursor()

    cursor.execute("SELECT car_id FROM rentals WHERE rental_id = %s", (rental_id,))
    car = cursor.fetchone()
    if car:
        car_id = car[0]
        update_car_query = "UPDATE cars SET status = 'available' WHERE car_id = %s"
        cursor.execute(update_car_query, (car_id,))

        delete_rental_query = "DELETE FROM rentals WHERE rental_id = %s"
        cursor.execute(delete_rental_query, (rental_id,))
        connection.commit()

        print("Mobil berhasil dikembalikan.")
    else:
        print("Data rental tidak ditemukan.")

# Fungsi untuk melihat mobil yang sedang disewa
def list_rented_cars(connection):
    cursor = connection.cursor()
    query = """
        SELECT rentals.rental_id, customers.name, cars.car_name, cars.plate_number, rentals.start_date, rentals.end_date
        FROM rentals
        JOIN customers ON rentals.customer_id = customers.customer_id
        JOIN cars ON rentals.car_id = cars.car_id
        WHERE cars.status = 'rented'
    """
    cursor.execute(query)
    result = cursor.fetchall()

    if cursor.rowcount <= 0:
        print("Tidak ada mobil yang sedang disewakan.")
    else:
        table = PrettyTable(["ID Rental", "Nama Pelanggan", "Nama Mobil", "Plat Nomor", "Mulai", "Selesai"])
        for rental in result:
            table.add_row(rental)
        print(table)

# Fungsi untuk menghapus data pelanggan
def delete_customer(connection):
    customer_id = input("Masukkan ID pelanggan yang akan dihapus: ")
    cursor = connection.cursor()

    # Hapus data pelanggan
    cursor.execute("DELETE FROM customers WHERE customer_id = %s", (customer_id,))
    connection.commit()

    # Reset Auto Increment ID untuk tabel customers
    cursor.execute("ALTER TABLE customers AUTO_INCREMENT = 1")
    connection.commit()

    print(f"{cursor.rowcount} pelanggan berhasil dihapus.")

# Fungsi untuk menghapus data mobil
def delete_car(connection):
    car_id = input("Masukkan ID mobil yang akan dihapus: ")
    cursor = connection.cursor()

    # Hapus data mobil
    cursor.execute("DELETE FROM cars WHERE car_id = %s", (car_id,))
    connection.commit()

    # Reset Auto Increment ID untuk tabel cars
    cursor.execute("ALTER TABLE cars AUTO_INCREMENT = 1")
    connection.commit()

    print(f"{cursor.rowcount} mobil berhasil dihapus.")

# Fungsi untuk menampilkan menu utama
def show_menu(connection):
    print_cool_header()
    print(Fore.GREEN + "1. Tambah Data Mobil")
    print(Fore.GREEN + "2. Lihat Data Mobil")
    print(Fore.GREEN + "3. Tambah Data Pelanggan")
    print(Fore.GREEN + "4. Lihat Data Pelanggan")
    print(Fore.GREEN + "5. Sewa Mobil")
    print(Fore.GREEN + "6. Kembalikan Mobil")
    print(Fore.GREEN + "7. Lihat Mobil yang Disewakan")
    print(Fore.GREEN + "8. Hapus Data Pelanggan")
    print(Fore.GREEN + "9. Hapus Data Mobil")
    print(Fore.RED + "0. Keluar\n")

    menu = input(Fore.CYAN + "Pilih Menu > ")

    if menu == "1":
        add_car(connection)
    elif menu == "2":
        show_cars(connection)
    elif menu == "3":
        add_customer(connection)
    elif menu == "4":
        show_customers(connection)
    elif menu == "5":
        rent_car(connection)
    elif menu == "6":
        return_car(connection)
    elif menu == "7":
        list_rented_cars(connection)
    elif menu == "8":
        delete_customer(connection)
    elif menu == "9":
        delete_car(connection)
    elif menu == "0":
        exit()
    else:
        print(Fore.RED + "Menu salah!")

if _name_ == "_main_":
    while True:
        show_menu(connection)
