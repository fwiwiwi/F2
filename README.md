using System;
using System.IO;
using System.Security.Cryptography;
using System.Text;

class FileStorageApp
{
    private string mainDirectory;
    private string mirrorDirectory;

    static void Main()
    {
        FileStorageApp app = new FileStorageApp();
        app.Run();
    }

    void Run()
    {
        Console.WriteLine("Приложение-хранилище файлов с зеркалированием");
        
        // Выбор каталогов
        Console.Write("Введите основной каталог: ");
        mainDirectory = Console.ReadLine();
        
        Console.Write("Введите каталог-зеркало: ");
        mirrorDirectory = Console.ReadLine();

        // Создание каталогов
        Directory.CreateDirectory(mainDirectory);
        Directory.CreateDirectory(mirrorDirectory);

        while (true)
        {
            Console.WriteLine("\n1. Загрузить файл");
            Console.WriteLine("2. Выгрузить файл");
            Console.WriteLine("3. Выход");
            Console.Write("Выберите действие: ");
            
            string choice = Console.ReadLine();
            
            switch (choice)
            {
                case "1":
                    UploadFile();
                    break;
                case "2":
                    DownloadFile();
                    break;
                case "3":
                    return;
                default:
                    Console.WriteLine("Неверный выбор");
                    break;
            }
        }
    }

    void UploadFile()
    {
        Console.Write("Введите путь к файлу для загрузки: ");
        string sourcePath = Console.ReadLine();
        
        if (!File.Exists(sourcePath))
        {
            Console.WriteLine("Файл не существует!");
            return;
        }

        string fileName = Path.GetFileName(sourcePath);
        string mainPath = Path.Combine(mainDirectory, fileName);
        string mirrorPath = Path.Combine(mirrorDirectory, fileName);

        try
        {
            // Копирование в оба каталога
            File.Copy(sourcePath, mainPath, true);
            File.Copy(sourcePath, mirrorPath, true);
            
            // Создание файлов с чек-суммами
            string checksum = CalculateChecksum(sourcePath);
            File.WriteAllText(mainPath + ".checksum", checksum);
            File.WriteAllText(mirrorPath + ".checksum", checksum);
            
            Console.WriteLine("Файл успешно загружен и зеркалирован.");
        }
        catch (Exception e)
        {
            Console.WriteLine($"Ошибка при загрузке: {e.Message}");
        }
    }

    void DownloadFile()
    {
        Console.Write("Введите имя файла для выгрузки: ");
        string fileName = Console.ReadLine();
        
        string mainPath = Path.Combine(mainDirectory, fileName);
        string mirrorPath = Path.Combine(mirrorDirectory, fileName);
        string mainChecksumPath = mainPath + ".checksum";
        string mirrorChecksumPath = mirrorPath + ".checksum";

        if (!File.Exists(mainPath) && !File.Exists(mirrorPath))
        {
            Console.WriteLine("Файл не существует в хранилище!");
            return;
        }

        try
        {
            string sourcePath = mainPath;
            string expectedChecksum = File.Exists(mainChecksumPath) ? 
                File.ReadAllText(mainChecksumPath) : null;
            
            // Проверка основного файла
            if (File.Exists(mainPath) && expectedChecksum != null)
            {
                string actualChecksum = CalculateChecksum(mainPath);
                if (actualChecksum != expectedChecksum)
                {
                    Console.WriteLine("Основной файл поврежден! Используем зеркало...");
                    sourcePath = mirrorPath;
                }
            }
            else if (!File.Exists(mainPath))
            {
                Console.WriteLine("Основного файла нет! Используем зеркало...");
                sourcePath = mirrorPath;
            }

            // Копирование файла пользователю
            Console.Write("Введите путь для сохранения: ");
            string savePath = Console.ReadLine();
            File.Copy(sourcePath, savePath, true);
            Console.WriteLine("Файл успешно выгружен.");
        }
        catch (Exception e)
        {
            Console.WriteLine($"Ошибка при выгрузке: {e.Message}");
        }
    }

    string CalculateChecksum(string filePath)
    {
        using var sha256 = SHA256.Create();
        using var stream = File.OpenRead(filePath);
        byte[] hash = sha256.ComputeHash(stream);
        return BitConverter.ToString(hash).Replace("-", "").ToLower();
    }
}
