using System;
using System.Collections.Generic;
using System.Threading;

namespace KnightAdventure
{
    abstract class Character
    {
        public string Name { get; protected set; }
        public int Health { get; protected set; }
        public int MaxHealth { get; protected set; }
        public int Stamina { get; protected set; }
        public int MaxStamina { get; protected set; }
        public int BaseDamage { get; protected set; }

        public bool IsAlive => Health > 0;

        public Character(string name, int health, int stamina, int damage)
        {
            Name = name;
            MaxHealth = health;
            Health = health;
            MaxStamina = stamina;
            Stamina = stamina;
            BaseDamage = damage;
        }

        public virtual void TakeDamage(int damage)
        {
            Health -= damage;
            if (Health < 0) Health = 0;
            Console.WriteLine($"-> {Name} получает {damage} урона. (HP: {Health}/{MaxHealth})");
        }

        public virtual void Attack(Character target)
        {
            Console.WriteLine($"⚔️ {Name} наносит базовый удар по {target.Name}!");
            target.TakeDamage(BaseDamage);
        }
    }

    class Knight : Character
    {
        public int Potions { get; private set; } = 3;
        public bool IsBlocking { get; set; } = false;

        public Knight(string name) : base(name, 150, 100, 20) { }

        // 1. Обычный удар (тратит мало выносливости)
        public override void Attack(Character target)
        {
            int staminaCost = 15;
            if (Stamina >= staminaCost)
            {
                Stamina -= staminaCost;
                Console.WriteLine($"⚔️ {Name} делает быстрый выпад мечом! (Выносливость -{staminaCost})");
                target.TakeDamage(BaseDamage);
            }
            else
            {
                Console.WriteLine($"💨 {Name} слишком устал для атаки! Нанесен слабый удар.");
                target.TakeDamage(BaseDamage / 2);
            }
        }

        // 2. Сильный удар (наносит двойной урон, но требует много выносливости)
        public void HeavyAttack(Character target)
        {
            int staminaCost = 40;
            if (Stamina >= staminaCost)
            {
                Stamina -= staminaCost;
                Console.ForegroundColor = ConsoleColor.Magenta;
                Console.WriteLine($"💥 {Name} наносит сокрушительный круговой удар! (Выносливость -{staminaCost})");
                Console.ResetColor();
                target.TakeDamage(BaseDamage * 2);
            }
            else
            {
                Console.WriteLine("❌ Недостаточно выносливости для сильного удара!");
                Attack(target); // Автоматически бьем обычным ударом
            }
        }

        // 3. Защита (снижает урон в следующем ходу и копит выносливость)
        public void Block()
        {
            IsBlocking = true;
            int staminaRegen = 25;
            Stamina = Math.Min(MaxStamina, Stamina + staminaRegen);
            Console.WriteLine($"🛡️ {Name} встал в оборонительную стойку. Урон снижен, выносливость восстановлена (+{staminaRegen})!");
        }

        // 4. Лечение
        public void Heal()
        {
            if (Potions > 0)
            {
                Health = Math.Min(MaxHealth, Health + 60);
                Stamina = Math.Min(MaxStamina, Stamina + 30);
                Potions--;
                Console.WriteLine($"✨ {Name} выпил зелье! Восстановлено 60 HP и 30 выносливости. (Осталось зелий: {Potions})");
            }
            else
            {
                Console.WriteLine("❌ Зелья закончились!");
            }
        }

        public override void TakeDamage(int damage)
        {
            if (IsBlocking)
            {
                damage /= 3; // Урон режется в 3 раза при блоке
                IsBlocking = false; // Сбрасываем блок после получения удара
                Console.Write("🛡️ Блок сработал! ");
            }
            base.TakeDamage(damage);
        }

        public void RegenerateStamina()
        {
            int regen = 10;
            Stamina = Math.Min(MaxStamina, Stamina + regen);
        }
    }

    class Enemy : Character
    {
        public Enemy(string name, int health, int stamina, int damage) : base(name, health, stamina, damage) { }

        public override void Attack(Character target)
        {
            // Логика врага: если есть выносливость, он бьет больно
            if (Stamina >= 20)
            {
                Stamina -= 20;
                base.Attack(target);
            }
            else
            {
                Console.WriteLine($"💤 {Name} тяжело дышит и атакует вполсилы.");
                Stamina += 15; // Отдыхает во время слабой атаки
                target.TakeDamage(BaseDamage / 2);
            }
        }
    }

    class Program
    {
        static void Main(string[] args)
        {
            Console.OutputEncoding = System.Text.Encoding.UTF8;
            
            Console.WriteLine("==================================================");
            Console.WriteLine("📜 Рыцарь, Твое приключение начинается! 📜");
            Console.WriteLine("==================================================\n");

            Console.Write("Введите имя рыцаря: ");
            string knightName = Console.ReadLine();
            if (string.IsNullOrWhiteSpace(knightName)) knightName = "Сир Галахад";

            Knight player = new Knight(knightName);

            Queue<Enemy> waves = new Queue<Enemy>();
            waves.Enqueue(new Enemy("Гоблин-разведчик 🎯", 50, 40, 15));
            waves.Enqueue(new Enemy("Орочий вождь 🪓", 100, 60, 25));
            waves.Enqueue(new Enemy("Древний Дракон 🐉", 200, 100, 40));

            int waveNumber = 1;

            while (player.IsAlive && waves.Count > 0)
            {
                Enemy currentEnemy = waves.Dequeue();
                Console.ForegroundColor = ConsoleColor.Yellow;
                Console.WriteLine($"\n⚔️ ВОЛНА {waveNumber}: Перед вами {currentEnemy.Name}! ⚔️");
                Console.ResetColor();
                Thread.Sleep(1000);

                while (player.IsAlive && currentEnemy.IsAlive)
                {
                    // Пассивное восстановление выносливости рыцаря в начале хода
                    player.RegenerateStamina();

                    Console.WriteLine($"\n--- Ход {player.Name} | ❤️ HP: {player.Health}/{player.MaxHealth} | ⚡ Выносливость: {player.Stamina}/{player.MaxStamina} ---");
                    Console.WriteLine("1. Быстрый удар (15 выносливости)");
                    Console.WriteLine("2. Сильный удар (40 выносливости)");
                    Console.WriteLine("3. Защита (+Выносливость, Снижение урона)");
                    Console.WriteLine($"4. Выпить зелье ({player.Potions} шт.)");
                    Console.Write("Ваш выбор: ");

                    string choice = Console.ReadLine();
                    Console.WriteLine();

                    switch (choice)
                    {
                        case "1":
                            player.Attack(currentEnemy);
                            break;
                        case "2":
                            player.HeavyAttack(currentEnemy);
                            break;
                        case "3":
                            player.Block();
                            break;
                        case "4":
                            player.Heal();
                            break;
                        default:
                            Console.WriteLine("🤔 Вы замешкались и пропустили ход!");
                            break;
                    }

                    // Ход врага
                    if (currentEnemy.IsAlive)
                    {
                        Thread.Sleep(800);
                        Console.ForegroundColor = ConsoleColor.Red;
                        Console.WriteLine($"\n--- Ход противника ({currentEnemy.Name}) ---");
                        currentEnemy.Attack(player);
                        Console.ResetColor();
                    }
                    Thread.Sleep(800);
                }

                if (player.IsAlive)
                {
                    Console.ForegroundColor = ConsoleColor.Green;
                    Console.WriteLine($"\n🎉 {currentEnemy.Name} повержен!");
                    Console.ResetColor();
                    waveNumber++;
                    Thread.Sleep(1500);
                }
            }

            Console.WriteLine("\n==================================================");
            if (player.IsAlive)
            {
                Console.ForegroundColor = ConsoleColor.Cyan;
                Console.WriteLine($"🏆 Победа! {player.Name} одолел все угрозы и стал легендой!");
                Console.ResetColor();
            }
            else
            {
                Console.ForegroundColor = ConsoleColor.DarkRed;
                Console.WriteLine("💀 Доспехи покрылись ржавчиной, а имя рыцаря забыто... Вы погибли.");
                Console.ResetColor();
            }
            Console.WriteLine("==================================================");
            Console.ReadKey();
        }
    }
}
