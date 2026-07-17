# CAPSTONE-Kontakt-kitobi-fayl-bilan-1. main.py (Dastur kodi)
Ushbu kodni main.py fayliga saqlang. Dasturda ism bo'sh bo'lmasligi va telefon raqami faqat raqamlardan iborat (yoki + bilan boshlanadigan) bo'lishi tekshiriladi (validatsiya).

Python
import json
import csv
import os
from datetime import datetime

# --- LOG FUNKSIYASI ---
def log_action(action: str):
    """Barcha amallarni vaqt bilan log.txt fayliga yozib boradi."""
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    try:
        with open("log.txt", "a", encoding="utf-8") as f:
            f.write(f"[{timestamp}] {action}\n")
    except Exception as e:
        print(f"Log yozishda xatolik: {e}")

# --- VALIDATSIYA XATOLIKLARI ---
class ValidationError(Exception):
    """Ma'lumotlar noto'g'ri kiritilganda chiquvchi xatolik klasi."""
    pass

# --- CONTACT KLASI ---
class Contact:
    def __init__(self, ism: str, telefon: str, email: str, manzil: str):
        self.ism = self.validate_ism(ism)
        self.telefon = self.validate_telefon(telefon)
        self.email = email
        self.manzil = manzil

    @staticmethod
    def validate_ism(ism: str) -> str:
        clean_ism = ism.strip()
        if not clean_ism:
            raise ValidationError("Ism bo'sh bo'lishi mumkin emas!")
        return clean_ism

    @staticmethod
    def validate_telefon(telefon: str) -> str:
        clean_tel = telefon.strip()
        # Telefon faqat raqamlar yoki + bilan boshlanib raqamlardan iborat bo'lishi kerak
        if not clean_tel:
            raise ValidationError("Telefon raqami bo'sh bo'lishi mumkin emas!")
        if clean_tel.startswith('+'):
            check_part = clean_tel[1:]
        else:
            check_part = clean_tel
        
        if not check_part.isdigit():
            raise ValidationError("Telefon raqami faqat raqamlardan (yoki + belgisidan) iborat bo'lishi kerak!")
        return clean_tel

    def to_dict(self):
        return {
            "ism": self.ism,
            "telefon": self.telefon,
            "email": self.email,
            "manzil": self.manzil
        }

# --- CONTACTBOOK KLASI ---
class ContactBook:
    def __init__(self, filename="kontaktlar.json"):
        self.filename = filename
        self.contacts = []
        self.load_contacts()

    def load_contacts(self):
        """JSON fayldan kontaktlarni xavfsiz yuklaydi."""
        if not os.path.exists(self.filename):
            self.contacts = []
            return
        try:
            with open(self.filename, "r", encoding="utf-8") as f:
                data = json.load(f)
                self.contacts = [Contact(**item) for item in data]
            log_action("Kontaktlar JSON fayldan muvaffaqiyatli yuklandi.")
        except json.JSONDecodeError:
            print("Xato: JSON fayli buzilgan. Bo'sh ro'yxat yaratildi.")
            log_action("Xato: JSON fayli buzilgan, yuklab bo'lmadi.")
            self.contacts = []
        except Exception as e:
            print(f"Kontaktlarni yuklashda kutilmagan xato: {e}")
            log_action(f"Yuklashda kutilmagan xato: {str(e)}")
            self.contacts = []

    def save_contacts(self):
        """Kontaktlarni JSON faylga saqlaydi."""
        try:
            with open(self.filename, "w", encoding="utf-8") as f:
                json.dump([c.to_dict() for c in self.contacts], f, indent=4, ensure_ascii=False)
        except Exception as e:
            print(f"Saqlashda xatolik yuz berdi: {e}")
            log_action(f"Saqlashda xato: {str(e)}")

    def qoshish(self, contact: Contact):
        # Bir xil telefon raqamli kontakt borligini tekshirish
        for c in self.contacts:
            if c.telefon == contact.telefon:
                raise ValidationError(f"Bu telefon raqami ({contact.telefon}) allaqachon mavjud!")
        
        self.contacts.append(contact)
        self.save_contacts()
        log_action(f"Yangi kontakt qo'shildi: {contact.ism} ({contact.telefon})")
        print(f"\n{contact.ism} muvaffaqiyatli qo'shildi!")

    def ochirish(self, telefon: str) -> bool:
        for c in self.contacts:
            if c.telefon == telefon:
                self.contacts.remove(c)
                self.save_contacts()
                log_action(f"Kontakt o'chirildi: {c.ism} ({telefon})")
                return True
        return False

    def qidirish(self, kalit_soz: str):
        """Ism yoki telefon bo'yicha qidiradi."""
        results = []
        for c in self.contacts:
            if kalit_soz.lower() in c.ism.lower() or kalit_soz in c.telefon:
                results.append(c)
        return results

    def yangilash(self, eski_tel: str, yangi_ism: str, yangi_tel: str, yangi_email: str, yangi_manzil: str) -> bool:
        # Avval eski kontaktni topamiz
        target_contact = None
        for c in self.contacts:
            if c.telefon == eski_tel:
                target_contact = c
                break
        
        if not target_contact:
            return False

        # Yangi ma'lumotlarni validatsiya qilamiz
        validated_ism = Contact.validate_ism(yangi_ism)
        validated_tel = Contact.validate_telefon(yangi_tel)

        # Agar telefon o'zgarayotgan bo'lsa, yangi telefon band emasligini tekshiramiz
        if validated_tel != eski_tel:
            for c in self.contacts:
                if c.telefon == validated_tel:
                    raise ValidationError(f"Yangi telefon raqami ({validated_tel}) allaqachon boshqa kontaktda band!")

        # Ma'lumotlarni yangilaymiz
        target_contact.ism = validated_ism
        target_contact.telefon = validated_tel
        target_contact.email = yangi_email
        target_contact.manzil = yangi_manzil

        self.save_contacts()
        log_action(f"Kontakt yangilandi: {validated_ism} ({validated_tel})")
        return True

    def list_all(self):
        return self.contacts

    def export_to_csv(self, csv_filename="kontaktlar.csv"):
        """Bonus: Barcha kontaktlarni CSV formatiga eksport qiladi."""
        try:
            with open(csv_filename, "w", newline="", encoding="utf-8") as f:
                writer = csv.writer(f)
                writer.writerow(["Ism", "Telefon", "Email", "Manzil"])
                for c in self.contacts:
                    writer.writerow([c.ism, c.telefon, c.email, c.manzil])
            log_action(f"Kontaktlar CSV ga eksport qilindi: {csv_filename}")
            print(f"\nKontaktlar '{csv_filename}' fayliga muvaffaqiyatli eksport qilindi!")
        except Exception as e:
            print(f"CSV eksportida xatolik: {e}")
            log_action(f"CSV eksport xatosi: {str(e)}")


# --- FOYDALANUVCHI INTERFEYSI (MENU) ---
def main():
    book = ContactBook()

    while True:
        print("\n" + "="*35)
        print("     KONTAKTLAR KITOBI MENU")
        print("="*35)
        print("1. Yangi kontakt qo'shish")
        print("2. Barcha kontaktlarni ko'rish")
        print("3. Kontakt qidirish")
        print("4. Kontaktni yangilash")
        print("5. Kontaktni o'chirish")
        print("6. CSV ga eksport qilish (Bonus)")
        print("0. Chiqish")
        print("="*35)
        
        tanlov = input("Amalni tanlang (0-6): ").strip()

        if tanlov == "1":
            print("\n--- Yangi kontakt yaratish ---")
            ism = input("Ism kiriting: ")
            tel = input("Telefon kiriting: ")
            email = input("Email kiriting: ")
            manzil = input("Manzil kiriting: ")
            try:
                # Validatsiya obyekt yaratilish jarayonida tekshiriladi
                yangi_kontakt = Contact(ism, tel, email, manzil)
                book.qoshish(yangi_kontakt)
            except ValidationError as ve:
                print(f"\n❌ Validatsiya xatosi: {ve}")
                log_action(f"Muvaffaqiyatsiz qo'shish urinishi: {ve}")
            except Exception as e:
                print(f"\n❌ Kutilmagan xato: {e}")

        elif tanlov == "2":
            kontaktlar = book.list_all()
            print("\n--- Barcha kontaktlar ro'yxati ---")
            if not kontaktlar:
                print("Kontaktlar ro'yxati bo'sh.")
            else:
                for idx, c in enumerate(kontaktlar, 1):
                    print(f"{idx}. {c.ism} | Tel: {c.telefon} | Email: {c.email} | Manzil: {c.manzil}")

        elif tanlov == "3":
            print("\n--- Kontakt qidirish ---")
            kalit = input("Ism yoki telefon qismini kiriting: ").strip()
            natijalar = book.qidirish(kalit)
            if not natijalar:
                print("Hech qanday kontakt topilmadi.")
            else:
                print(f"\nTopilgan kontaktlar ({len(natijalar)} ta):")
                for idx, c in enumerate(natijalar, 1):
                    print(f"{idx}. {c.ism} | Tel: {c.telefon} | Email: {c.email} | Manzil: {c.manzil}")

        elif tanlov == "4":
            print("\n--- Kontaktni yangilash ---")
            eski_tel = input("Yangilanmoqchi bo'lgan kontaktning telefon raqamini kiriting: ").strip()
            
            # Avval shu raqamli kontakt borligini tekshiramiz
            topildi = book.qidirish(eski_tel)
            mos_kontakt = [c for c in topildi if c.telefon == eski_tel]
            
            if not mos_kontakt:
                print("Bunday telefon raqamli kontakt topilmadi!")
                continue
            
            print(f"Topildi: {mos_kontakt[0].ism} ({mos_kontakt[0].telefon})")
            yangi_ism = input("Yangi ism kiriting: ")
            yangi_tel = input("Yangi telefon kiriting: ")
            yangi_email = input("Yangi email kiriting: ")
            yangi_manzil = input("Yangi manzil kiriting: ")

            try:
                if book.yangilash(eski_tel, yangi_ism, yangi_tel, yangi_email, yangi_manzil):
                    print("\nKontakt muvaffaqiyatli yangilandi!")
                else:
                    print("\nYangilashda xatolik yuz berdi.")
            except ValidationError as ve:
                print(f"\n❌ Validatsiya xatosi: {ve}")
                log_action(f"Muvaffaqiyatsiz yangilash urinishi: {ve}")

        elif tanlov == "5":
            print("\n--- Kontaktni o'chirish ---")
            tel = input("O'chiriladigan kontakt telefon raqamini kiriting: ").strip()
            if book.ochirish(tel):
                print("\nKontakt muvaffaqiyatli o'chirildi!")
            else:
                print("\nBunday telefon raqamli kontakt topilmadi.")

        elif tanlov == "6":
            book.export_to_csv()

        elif tanlov == "0":
            print("\nDastur tugatildi. Kuningiz xayrli o'tsin!")
            log_action("Dastur foydalanuvchi tomonidan yopildi.")
            break
        else:
            print("\nNoto'g'ri tanlov! Iltimos, 0 dan 6 gacha bo'lgan raqam kiriting.")

if __name__ == "__main__":
    log_action("Dastur ishga tushirildi.")
    main()
2. README.md (Loyiha Hujjati)
Ushbu matnni README.md nomli alohida faylga saqlang.

Markdown
# 📇 Kontaktlar Kitobi (Contact Book)

Ushbu dastur konsol orqali ishlaydigan va kontaktlarni boshqarish imkonini beruvchi interaktiv dasturdir. Ma'lumotlar **JSON** formatida saqlanadi, har bir amal **log.txt** fayliga vaqti bilan yozib boriladi va ma'lumotlar **CSV** formatiga eksport qilinishi mumkin.

---

## ✨ Xususiyatlari
* **JSON saqlash tizimi:** Ma'lumotlar `kontaktlar.json` faylida saqlanadi.
* **Xatoliklarni boshqarish:** Fayl yuklash yoki yozishda `try-except` bloklari orqali xatoliklar xavfsiz boshqariladi.
* **Ma'lumotlar Validatsiyasi:** Bo'sh ism va noto'g'ri telefon raqamlari qabul qilinmaydi (`ValidationError` qaytariladi).
* **Logging tizimi:** Har bir amal (qo'shish, o'chirish, xatoliklar) `log.txt` fayliga avtomatik tarzda qayd etiladi.
* **Bonus - CSV Eksport:** Barcha kontaktlarni birgina buyruq bilan `kontaktlar.csv` fayliga ko'chirish mumkin.

---

## 🚀 Ishga tushirish

Dasturni ishga tushirish uchun kompyuteringizda **Python 3.x** o'rnatilgan bo'lishi lozim.

1. Loyiha fayllari joylashgan papkaga kiring.
2. Terminal yoki buyruqlar satrida (CMD) quyidagi buyruqni bajaring:

```bash
python main.py
📸 Foydalanishdan namuna (Screenshot)
Plaintext
===================================
     KONTAKTLAR KITOBI MENU
===================================
1. Yangi kontakt qo'shish
2. Barcha kontaktlarni ko'rish
3. Kontakt qidirish
4. Kontaktni yangilash
5. Kontaktni o'chirish
6. CSV ga eksport qilish (Bonus)
0. Chiqish
===================================
Amalni tanlang (0-6): 1

--- Yangi kontakt yaratish ---
Ism kiriting: Alisher
Telefon kiriting: +998901234567
Email kiriting: alisher@mail.com
Manzil kiriting: Toshkent, Chilonzor

Alisher muvaffaqiyatli qo'shildi!
Noto'g'ri kiritish holatida validatsiya xatosi:
Plaintext
Amalni tanlang (0-6): 1

--- Yangi kontakt yaratish ---
Ism kiriting: 
Telefon kiriting: abcde
Email kiriting: test@mail.com
Manzil kiriting: Samarqand

❌ Validatsiya xatosi: Ism bo'sh bo'lishi mumkin emas!
