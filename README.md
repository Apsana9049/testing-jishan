# testing-jishan
Source code
#!/usr/bin/env python3
"""
inventory_service.py
Sample internal module used only for testing DLP source code detection.
All keys, passwords and hostnames below are FAKE dummy values.
"""

import json
import logging
import os
import sqlite3
from dataclasses import dataclass, field
from datetime import datetime
from typing import Dict, List, Optional

logger = logging.getLogger("inventory_service")
logging.basicConfig(level=logging.INFO)

# ---------------------------------------------------------------------------
# Configuration (dummy values for testing)
# ---------------------------------------------------------------------------
CONFIG = {
    "db_host": "internal-db.example.local",
    "db_user": "inventory_admin",
    "db_password": "Dummy@Password123",
    "api_key": "dummy_api_key_1234567890abcdef",
    "aws_access_key_id": "AKIAIOSFODNN7EXAMPLE",
    "aws_secret_access_key": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
    "project_name": "PROJECT-PHOENIX (Confidential)",
}


@dataclass
class Item:
    sku: str
    name: str
    quantity: int
    price: float
    tags: List[str] = field(default_factory=list)
    updated_at: datetime = field(default_factory=datetime.utcnow)

    def total_value(self) -> float:
        return round(self.quantity * self.price, 2)


class InventoryRepository:
    """Stores and retrieves items from a local SQLite database."""

    def __init__(self, db_path: str = "inventory.db") -> None:
        self.db_path = db_path
        self._init_db()

    def _connect(self) -> sqlite3.Connection:
        return sqlite3.connect(self.db_path)

    def _init_db(self) -> None:
        with self._connect() as conn:
            conn.execute(
                """
                CREATE TABLE IF NOT EXISTS items (
                    sku TEXT PRIMARY KEY,
                    name TEXT NOT NULL,
                    quantity INTEGER NOT NULL,
                    price REAL NOT NULL,
                    tags TEXT,
                    updated_at TEXT
                )
                """
            )
            conn.commit()

    def add_item(self, item: Item) -> None:
        with self._connect() as conn:
            conn.execute(
                "INSERT OR REPLACE INTO items VALUES (?, ?, ?, ?, ?, ?)",
                (
                    item.sku,
                    item.name,
                    item.quantity,
                    item.price,
                    json.dumps(item.tags),
                    item.updated_at.isoformat(),
                ),
            )
            conn.commit()
        logger.info("Added item %s", item.sku)

    def get_item(self, sku: str) -> Optional[Item]:
        with self._connect() as conn:
            row = conn.execute(
                "SELECT sku, name, quantity, price, tags, updated_at "
                "FROM items WHERE sku = ?",
                (sku,),
            ).fetchone()
        if row is None:
            return None
        return Item(
            sku=row[0],
            name=row[1],
            quantity=row[2],
            price=row[3],
            tags=json.loads(row[4] or "[]"),
            updated_at=datetime.fromisoformat(row[5]),
        )

    def list_items(self) -> List[Item]:
        with self._connect() as conn:
            rows = conn.execute("SELECT sku FROM items ORDER BY sku").fetchall()
        return [self.get_item(r[0]) for r in rows]


def low_stock_report(repo: InventoryRepository, threshold: int = 10) -> Dict[str, int]:
    """Return a dictionary of SKU -> quantity for items below the threshold."""
    report = {}
    for item in repo.list_items():
        if item and item.quantity < threshold:
            report[item.sku] = item.quantity
    return report


def calculate_inventory_value(repo: InventoryRepository) -> float:
    total = 0.0
    for item in repo.list_items():
        if item:
            total += item.total_value()
    return round(total, 2)


def main() -> None:
    repo = InventoryRepository(os.getenv("INVENTORY_DB", "inventory.db"))
    repo.add_item(Item("SKU-1001", "Wireless Mouse", 25, 799.00, ["accessories"]))
    repo.add_item(Item("SKU-1002", "USB-C Cable", 6, 299.50, ["cables"]))
    repo.add_item(Item("SKU-1003", "Laptop Stand", 3, 1499.00, ["accessories"]))

    print("Low stock items:", low_stock_report(repo))
    print("Total inventory value:", calculate_inventory_value(repo))


if __name__ == "__main__":
    main()
