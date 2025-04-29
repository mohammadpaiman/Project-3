import json
import csv
import requests
import os
import matplotlib.pyplot as plt

# Global variables
inventory_file = "tu_vending_inventory.json"
transaction_file = "transactions.csv"
exchange_api_url = "https://api.exchangerate.host/latest?base=USD"
current_currency = "USD"
currency_rates = {"USD": 1.0}
slots = []
sales_total = 0.0
transactions = []

def loadInventory():
    """Load inventory from the JSON file and assign slot IDs."""
    with open(inventory_file, "r") as f:
        data = json.load(f)
    inventory = {}
    rows = ['A', 'B', 'C', 'D', 'E', 'F']
    slot_num = 0
    for row in rows:
        for col in range(1, 6):
            if slot_num < len(data['inventory']):
                slot_id = f"{row}{col}"
                item = data['inventory'][slot_num]
                inventory[slot_id] = item
                slot_num += 1
    return inventory

def saveEndingInventory(my_inventory):
    """Save ending inventory back to JSON."""
    items = []
    for slot, item in my_inventory.items():
        items.append(item)
    with open("ending_inventory.json", "w") as f:
        json.dump({"inventory": items}, f, indent=4)

def saveTransaction():
    """Save all transactions to a CSV file."""
    with open(transaction_file, "w", newline='') as csvfile:
        writer = csv.writer(csvfile)
        writer.writerow(["SLOT_ID", "AMOUNT_USD", "ITEM"])
        for txn in transactions:
            writer.writerow(txn)

def displayInventory(my_inventory):
    """Display inventory with current currency."""
    print("---- Current vending machine inventory ----")
    for slot, item in my_inventory.items():
        if item['quantity'] > 0:
            price = round(item['price_usd'] * currency_rates[current_currency], 2)
            print(f"{slot}: {current_currency} {price} - {item['item']} ({item['quantity']} left)")
        else:
            print(f"{slot}: SOLD OUT - {item['item']}")
    print("--------------------------------------------")

def decrementInventory(my_inventory, slot_id):
    """Reduce the quantity of an item after a purchase."""
    if slot_id in my_inventory:
        if my_inventory[slot_id]['quantity'] > 0:
            my_inventory[slot_id]['quantity'] -= 1
            return True
        else:
            print("Sorry, item is sold out!")
            return False
    else:
        print("Invalid slot.")
        return False

def purchaseItem(my_inventory, slot_id):
    """Handle the purchase of an item."""
    global sales_total
    if slot_id in my_inventory:
        item = my_inventory[slot_id]
        if item['quantity'] > 0:
            price_usd = item['price_usd']
            price_currency = round(price_usd * currency_rates[current_currency], 2)
            print(f"You have purchased {item['item']} for {current_currency} {price_currency}. Thank you!")
            decrementInventory(my_inventory, slot_id)
            transactions.append([slot_id, price_usd, item['item']])
            sales_total += price_usd
        else:
            print("Item is out of stock.")
    else:
        print("Invalid slot.")

def changeCurrency():
    """Allow user to switch currency."""
    global current_currency
    print("Available currencies: USD (default), CAD, EUR, JPY")
    new_currency = input("Enter new currency (leave blank to cancel): ").upper().strip()
    if new_currency in ["USD", "CAD", "EUR", "JPY"]:
        updateExchangeRates()
        if new_currency in currency_rates:
            current_currency = new_currency
            print(f"Currency changed to {current_currency}.")
        else:
            print(f"Currency {new_currency} not available right now.")
    elif new_currency == "":
        print("Currency change cancelled.")
    else:
        print("Invalid currency.")

def updateExchangeRates():
    """Fetch exchange rates from the API."""
    global currency_rates
    try:
        response = requests.get(exchange_api_url)
        if response.status_code == 200:
            data = response.json()
            rates = data['rates']
            currency_rates['USD'] = 1.0
            currency_rates['CAD'] = rates.get('CAD', 1.0)
            currency_rates['EUR'] = rates.get('EUR', 1.0)
            currency_rates['JPY'] = rates.get('JPY', 1.0)
        else:
            print("Failed to fetch exchange rates.")
    except:
        print("Error connecting to exchange rate API.")

def generateInventoryChart(my_inventory):
    """Generate and display a bar chart of current inventory."""
    items = [item['item'] for item in my_inventory.values()]
    quantities = [item['quantity'] for item in my_inventory.values()]
    plt.bar(items, quantities)
    plt.xticks(rotation=90)
    plt.title('Vending Machine Inventory Levels')
    plt.xlabel('Item')
    plt.ylabel('Quantity')
    plt.tight_layout()
    plt.show()

def printMenu():
    """Display menu options."""
    print("\nPlease choose from the following options:")
    print(" (d) Display items")
    print(" (p) Purchase item")
    print(" ($) Change currency")
    print(" (c) Generate inventory chart")
    print(" (q) Quit")

def main():
    """Main vending machine loop."""
    global sales_total
    my_inventory = loadInventory()
    print("Welcome to the TU Vending Machine!")
    keepLooping = True
    while keepLooping:
        printMenu()
        choice = input(">").lower().strip()
        if choice == "d":
            displayInventory(my_inventory)
        elif choice == "p":
            slot = input("Enter slot ID to purchase: ").upper().strip()
            purchaseItem(my_inventory, slot)
        elif choice == "$":
            changeCurrency()
        elif choice == "c":
            generateInventoryChart(my_inventory)
        elif choice == "q":
            keepLooping = False
        else:
            print("Invalid option. Please try again.")
    print(f"Thank you for vending with us! Total collected: USD ${sales_total:.2f}")
    saveEndingInventory(my_inventory)
    saveTransaction()

#  UNIT TESTS 

def test_decrementInventory():
    """Test decrementInventory reduces quantity."""
    test_inv = {"A1": {"item": "TestItem", "quantity": 1, "price_usd": 1.0}}
    result = decrementInventory(test_inv, "A1")
    assert result == True
    assert test_inv["A1"]["quantity"] == 0

def test_purchaseItem():
    """Test purchaseItem logs transaction and updates sales_total."""
    global sales_total
    sales_total = 0.0
    test_inv = {"B1": {"item": "TestItem", "quantity": 1, "price_usd": 2.0}}
    purchaseItem(test_inv, "B1")
    assert sales_total == 2.0
    assert test_inv["B1"]["quantity"] == 0

if __name__ == "__main__":
    main()
