using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;

// Define the structure for an expense record
public class Expense
{
    public string Category { get; set; }
    public string Payee { get; set; }
    public decimal Amount { get; set; }
    public DateTime Date { get; set; }

    // Override ToString to provide a readable summary
    public override string ToString()
    {
        return $"{Date.ToShortDateString(),-10} | {Category,-15} | {Payee,-20} | {Amount,10:C2}";
    }

    // Static method to convert an Expense object to a CSV string for saving
    public static string ToCsv(Expense expense)
    {
        return $"{expense.Date:yyyy-MM-dd},{expense.Category},{expense.Payee},{expense.Amount}";
    }

    // Static method to parse a CSV string back into an Expense object
    public static Expense FromCsv(string csvLine)
    {
        string[] values = csvLine.Split(',');

        if (values.Length != 4)
        {
            throw new FormatException("CSV line does not contain 4 expected fields.");
        }

        // Parse date, category, payee, and amount
        return new Expense
        {
            Date = DateTime.Parse(values[0]),
            Category = values[1],
            Payee = values[2],
            Amount = decimal.Parse(values[3])
        };
    }
}

public class ExpenseTracker
{
    private const string DataFile = "expenses.txt";
    private List<Expense> expenses = new List<Expense>();

    public ExpenseTracker()
    {
        LoadExpenses(); // Load data when the tracker is initialized
    }

    // --- Data Persistence Methods ---

    // Load expenses from the text file
    private void LoadExpenses()
    {
        if (File.Exists(DataFile))
        {
            try
            {
                // Read all lines and use LINQ to parse each line into an Expense object
                expenses = File.ReadAllLines(DataFile)
                               .Where(line => !string.IsNullOrWhiteSpace(line)) // Skip empty lines
                               .Select(Expense.FromCsv)
                               .ToList();
                Console.WriteLine($"\nSuccessfully loaded {expenses.Count} expenses.");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"\nError loading expenses: {ex.Message}");
                expenses = new List<Expense>(); // Start with an empty list if loading fails
            }
        }
        else
        {
            Console.WriteLine("\nNo existing expense data file found. Starting fresh.");
        }
    }

    // Save current expenses to the text file
    private void SaveExpenses()
    {
        try
        {
            // Convert the list of Expense objects to a list of CSV strings and write to file
            List<string> csvLines = expenses.Select(Expense.ToCsv).ToList();
            File.WriteAllLines(DataFile, csvLines);
            Console.WriteLine("\nExpenses successfully saved.");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"\nError saving expenses: {ex.Message}");
        }
    }

    // --- Core Feature Methods ---

    public void AddNewExpense()
    {
        Console.WriteLine("\n--- Add New Expense ---");

        // 1. Get Category
        Console.Write("Enter Category (e.g., Food, Rent, Entertainment): ");
        string category = Console.ReadLine();
        if (string.IsNullOrWhiteSpace(category))
        {
            Console.WriteLine("Category cannot be empty. Aborting.");
            return;
        }

        // 2. Get Payee
        Console.Write("Enter Payee (e.g., Publix, Landlord, Cinema): ");
        string payee = Console.ReadLine();
        if (string.IsNullOrWhiteSpace(payee))
        {
            Console.WriteLine("Payee cannot be empty. Aborting.");
            return;
        }

        // 3. Get Amount (with validation)
        decimal amount = 0;
        bool validAmount = false;
        while (!validAmount)
        {
            Console.Write("Enter Amount (e.g., 7.13): $");
            if (decimal.TryParse(Console.ReadLine(), out amount) && amount > 0)
            {
                validAmount = true;
            }
            else
            {
                Console.WriteLine("Invalid amount. Please enter a positive number.");
            }
        }

        // 4. Get Date (default to today, or allow input)
        DateTime date = DateTime.Today;
        Console.Write($"Enter Date (YYYY-MM-DD) or press Enter for today ({date.ToShortDateString()}): ");
        string dateInput = Console.ReadLine();
        if (!string.IsNullOrWhiteSpace(dateInput) && DateTime.TryParse(dateInput, out DateTime inputDate))
        {
            date = inputDate;
        }

        // Create the new expense and add it to the list
        Expense newExpense = new Expense
        {
            Category = category.Trim(),
            Payee = payee.Trim(),
            Amount = amount,
            Date = date
        };

        expenses.Add(newExpense);
        SaveExpenses(); // Save immediately after adding
        Console.WriteLine("\nExpense added successfully!");
    }

    public void ViewAllExpenses()
    {
        Console.WriteLine("\n--- All Expenses ---");
        if (expenses.Count == 0)
        {
            Console.WriteLine("No expenses recorded yet.");
            return;
        }

        // Sort by date (most recent first)
        var sortedExpenses = expenses.OrderByDescending(e => e.Date).ToList();

        // Print header
        Console.WriteLine(new string('-', 60));
        Console.WriteLine($"{"Date",-10} | {"Category",-15} | {"Payee",-20} | {"Amount",10}");
        Console.WriteLine(new string('-', 60));

        // Print each expense
        foreach (var expense in sortedExpenses)
        {
            Console.WriteLine(expense);
        }

        // Print a summary total
        Console.WriteLine(new string('-', 60));
        Console.WriteLine($"TOTAL: {expenses.Sum(e => e.Amount),50:C2}");
        Console.WriteLine(new string('-', 60));
    }

    public void ViewPayees()
    {
        Console.WriteLine("\n--- Payee Summary ---");
        if (expenses.Count == 0)
        {
            Console.WriteLine("No expenses recorded yet.");
            return;
        }

        // Group expenses by Payee and calculate the total amount spent for each
        var payeeSummary = expenses
            .GroupBy(e => e.Payee)
            .Select(g => new { Payee = g.Key, TotalAmount = g.Sum(e => e.Amount) })
            .OrderByDescending(x => x.TotalAmount); // Sort by total amount

        // Print results
        Console.WriteLine(new string('-', 40));
        Console.WriteLine($"{"Payee",-25} | {"Total Spent",10}");
        Console.WriteLine(new string('-', 40));
        foreach (var summary in payeeSummary)
        {
            Console.WriteLine($"{summary.Payee,-25} | {summary.TotalAmount,10:C2}");
        }
        Console.WriteLine(new string('-', 40));
    }

    public void ViewCategories()
    {
        Console.WriteLine("\n--- Category Summary ---");
        if (expenses.Count == 0)
        {
            Console.WriteLine("No expenses recorded yet.");
            return;
        }

        // Group expenses by Category and calculate the total amount spent for each
        var categorySummary = expenses
            .GroupBy(e => e.Category)
            .Select(g => new { Category = g.Key, TotalAmount = g.Sum(e => e.Amount) })
            .OrderByDescending(x => x.TotalAmount); // Sort by total amount

        // Print results
        Console.WriteLine(new string('-', 40));
        Console.WriteLine($"{"Category",-25} | {"Total Spent",10}");
        Console.WriteLine(new string('-', 40));
        foreach (var summary in categorySummary)
        {
            Console.WriteLine($"{summary.Category,-25} | {summary.TotalAmount,10:C2}");
        }
        Console.WriteLine(new string('-', 40));
    }

    // --- Menu and Execution ---

    public void Run()
    {
        bool isRunning = true;
        while (isRunning)
        {
            Console.WriteLine("\n==============================");
            Console.WriteLine("💰 C# Expense Tracker 💰");
            Console.WriteLine("==============================");
            Console.WriteLine("1. Add New Expense");
            Console.WriteLine("2. View All Expenses");
            Console.WriteLine("3. View Payee Summary");
            Console.WriteLine("4. View Category Summary");
            Console.WriteLine("5. Exit and Save");
            Console.Write("Enter your choice (1-5): ");

            string choice = Console.ReadLine();

            // Using a switch statement for clear menu handling
            switch (choice)
            {
                case "1":
                    AddNewExpense();
                    break;
                case "2":
                    ViewAllExpenses();
                    break;
                case "3":
                    ViewPayees();
                    break;
                case "4":
                    ViewCategories();
                    break;
                case "5":
                    // SaveExpenses() is called immediately after adding an expense,
                    // but calling it here as a final step ensures all data is saved
                    // before the application exits.
                    SaveExpenses();
                    isRunning = false;
                    break;
                default:
                    Console.WriteLine("\nInvalid choice. Please enter a number between 1 and 5.");
                    break;
            }
        }
        Console.WriteLine("\nThank you for using the Expense Tracker. Goodbye!");
    }
}

class Program
{
    static void Main(string[] args)
    {
        ExpenseTracker tracker = new ExpenseTracker();
        tracker.Run();
    }
}
