import csv
import os
from datetime import datetime
import pandas as pd
import matplotlib.pyplot as plt

FILE_NAME = "transactions.csv"
HEADERS = ["Date", "Type", "Category", "Amount", "Description"]


def initialize_file():
    """Creates the CSV file with headers if missing or empty."""
    if not os.path.exists(FILE_NAME) or os.path.getsize(FILE_NAME) == 0:
        with open(FILE_NAME, mode="w", newline="", encoding="utf-8") as file:
            writer = csv.writer(file)
            writer.writerow(HEADERS)


def load_data():
    """Safely loads CSV data and ensures proper column headers exist."""
    initialize_file()
    df = pd.read_csv(FILE_NAME)
    df.columns = df.columns.str.strip().str.title()

    # If the CSV was saved without headers, reload using HEADERS
    if "Type" not in df.columns and len(df.columns) == 5:
        df = pd.read_csv(FILE_NAME, names=HEADERS)
        df.to_csv(FILE_NAME, index=False)

    df["Amount"] = pd.to_numeric(df["Amount"], errors="coerce").fillna(0)
    return df


def add_transaction():
    """Prompts user for transaction details, validates them, and saves to CSV."""
    print("\n--- Add New Transaction ---")

    # Validate Date
    while True:
        date_str = input("Enter date (YYYY-MM-DD) [Leave blank for today]: ").strip()
        if not date_str:
            date_str = datetime.today().strftime("%Y-%m-%d")
            break
        try:
            datetime.strptime(date_str, "%Y-%m-%d")
            break
        except ValueError:
            print("Invalid format. Please use YYYY-MM-DD.")

    # Validate Type
    while True:
        t_type = input("Type (Income / Expense): ").strip().capitalize()
        if t_type in ["Income", "Expense"]:
            break
        print("Invalid input. Enter 'Income' or 'Expense'.")

    # Category & Description
    category = input("Category (e.g., Food, Rent, Travel, Salary, Bills): ").strip().title()
    if not category:
        category = "General"

    # Validate Amount
    while True:
        try:
            amount = float(input("Amount (₹): ").strip())
            if amount > 0:
                break
            print("Amount must be greater than zero.")
        except ValueError:
            print("Invalid number. Please enter a valid amount.")

    description = input("Short description: ").strip()

    # Save to CSV
    with open(FILE_NAME, mode="a", newline="", encoding="utf-8") as file:
        writer = csv.writer(file)
        writer.writerow([date_str, t_type, category, amount, description])

    print("Transaction saved successfully!")


def view_recent_transactions():
    """Displays the last 10 transactions."""
    df = load_data()
    if df.empty:
        print("\nNo transactions recorded yet.")
    else:
        print("\n--- Recent Transactions ---")
        print(df.tail(10).to_string(index=False))


def view_summary():
    """Uses Pandas to compute budget balance, category totals, and monthly trends."""
    df = load_data()
    if df.empty:
        print("\nNo transactions found. Add some records first.")
        return

    print("\n================ FINANCIAL SUMMARY ================")

    # 1. Overall Budget Balance
    total_income = df[df["Type"] == "Income"]["Amount"].sum()
    total_expense = df[df["Type"] == "Expense"]["Amount"].sum()
    balance = total_income - total_expense

    print(f"Total Income   : ₹{total_income:,.2f}")
    print(f"Total Expenses : ₹{total_expense:,.2f}")
    print(f"Net Balance    : ₹{balance:,.2f}")

    # 2. Category-wise Expense Totals
    expenses_df = df[df["Type"] == "Expense"]
    if not expenses_df.empty:
        print("\n--- Expenses by Category ---")
        category_totals = expenses_df.groupby("Category")["Amount"].sum().sort_values(ascending=False)
        for cat, amt in category_totals.items():
            print(f"  {cat:<15}: ₹{amt:,.2f}")

    # 3. Monthly Trends
    df["Month"] = pd.to_datetime(df["Date"]).dt.to_period("M")
    monthly_summary = df.pivot_table(
        index="Month", columns="Type", values="Amount", aggfunc="sum", fill_value=0
    )
    print("\n--- Monthly Trends ---")
    print(monthly_summary)
    print("===================================================")


def generate_charts():
    """Generates Matplotlib visualizations for category breakdown and monthly trends."""
    df = load_data()
    if df.empty:
        print("\nNo data available to plot.")
        return

    expenses_df = df[df["Type"] == "Expense"]

    fig, axes = plt.subplots(1, 2, figsize=(13, 5))

    # Chart 1: Pie Chart of Expenses by Category
    if not expenses_df.empty:
        cat_data = expenses_df.groupby("Category")["Amount"].sum()
        axes[0].pie(cat_data, labels=cat_data.index, autopct="%1.1f%%", startangle=140)
        axes[0].set_title("Expense Breakdown by Category")
    else:
        axes[0].text(0.5, 0.5, "No Expense Data", ha="center", va="center")

    # Chart 2: Bar Chart of Monthly Income vs Expenses
    df["Month"] = pd.to_datetime(df["Date"]).dt.to_period("M").astype(str)
    monthly_data = df.pivot_table(
        index="Month", columns="Type", values="Amount", aggfunc="sum", fill_value=0
    )
    monthly_data.plot(kind="bar", ax=axes[1], color={"Income": "#2ecc71", "Expense": "#e74c3c"})
    axes[1].set_title("Monthly Income vs. Expenses")
    axes[1].set_xlabel("Month")
    axes[1].set_ylabel("Amount (₹)")
    axes[1].tick_params(axis="x", rotation=0)

    plt.tight_layout()
    plt.show()


def main():
    initialize_file()
    while True:
        print("\n=== PERSONAL EXPENSE TRACKER ===")
        print("1. Add Income / Expense")
        print("2. View Recent Transactions")
        print("3. View Budget Summary & Trends")
        print("4. Show Visual Charts")
        print("5. Exit")

        choice = input("Select an option (1-5): ").strip()

        if choice == "1":
            add_transaction()
        elif choice == "2":
            view_recent_transactions()
        elif choice == "3":
            view_summary()
        elif choice == "4":
            generate_charts()
        elif choice == "5":
            print("Exiting Expense Tracker. Goodbye!")
            break
        else:
            print("Invalid choice. Please enter a number between 1 and 5.")


if __name__ == "__main__":
    main()
