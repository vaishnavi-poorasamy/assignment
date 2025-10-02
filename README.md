package assignment;
import java.util.*;
import java.text.DecimalFormat;

public class RestaurantApp {
    private static final Scanner sc = new Scanner(System.in);
    private static final List<Table> tables = new ArrayList<>();
    private static final List<MenuItem> menu = new ArrayList<>();
    private static final List<Order> orders = new ArrayList<>();
    private static final List<KitchenTicket> tickets = new ArrayList<>();
    private static final List<Bill> bills = new ArrayList<>();
    private static int nextOrderId = 1;
    private static int nextTicketId = 1;
    private static int nextBillId = 1;
    private static final DecimalFormat df = new DecimalFormat("#0.00");

    public static void main(String[] args) {
        seedSampleData();

        boolean exit = false;
        while (!exit) {
            printMainMenu();
            int choice = readInt("Choose: ", 1, 9);
            switch (choice) {
                case 1 -> addTable();
                case 2 -> addMenuItem();
                case 3 -> seatCustomer();
                case 4 -> createOrder();
                case 5 -> sendToKitchen();
                case 6 -> generateBill();
                case 7 -> recordPayment();
                case 8 -> displayTablesAndMenu();
                case 9 -> {
                    System.out.println("Exiting. Goodbye!");
                    exit = true;
                }
            }
        }
    }

    // ========== Menu Actions ==========
    private static void addTable() {
        int id = tables.size() + 1;
        int seats = readInt("Enter number of seats for new table: ", 1, 20);
        tables.add(new Table(id, seats));
        System.out.println("Added Table #" + id + " with " + seats + " seats.");
    }

    private static void addMenuItem() {
        String id;
        while (true) {
            System.out.print("Enter menu item ID (e.g., M001): ");
            id = sc.nextLine().trim();
            if (id.isEmpty()) continue;
            if (findMenuItemById(id) != null) {
                System.out.println("Item already exists.");
            } else break;
        }
        System.out.print("Enter name: ");
        String name = sc.nextLine().trim();
        double price = readDouble("Enter price: ", 0.01, 10000);
        menu.add(new MenuItem(id, name, price));
        System.out.println("Menu item added: " + id + " - " + name + " ($" + df.format(price) + ")");
    }

    private static void seatCustomer() {
        for (Table t : tables) System.out.println(t);
        int tableId = readInt("Enter table id to seat customer: ", 1, tables.size());
        Table t = findTableById(tableId);
        if (t == null || t.isOccupied()) {
            System.out.println("Invalid or occupied table.");
            return;
        }
        System.out.print("Enter customer name: ");
        String name = sc.nextLine().trim();
        Customer c = new Customer(name, "");
        t.seatCustomer(c);
        System.out.println("Seated " + name + " at Table #" + t.getId());
    }

    private static void createOrder() {
        System.out.println("1. Dine-in\n2. Takeaway");
        int type = readInt("Choose: ", 1, 2);
        Order order;
        if (type == 1) {
            int tid = readInt("Enter table id: ", 1, tables.size());
            Table t = findTableById(tid);
            if (t == null || !t.isOccupied()) {
                System.out.println("Invalid or empty table.");
                return;
            }
            order = new Order(nextOrderId++, tid, false);
        } else {
            order = new Order(nextOrderId++, -1, true);
        }
        while (true) {
            displayMenu();
            System.out.print("Enter menu item id (or 'done'): ");
            String mid = sc.nextLine().trim();
            if (mid.equalsIgnoreCase("done")) break;
            MenuItem mi = findMenuItemById(mid);
            if (mi == null) { System.out.println("Not found."); continue; }
            int qty = readInt("Quantity: ", 1, 20);
            order.addItem(new OrderItem(mi, qty));
        }
        if (order.getItems().isEmpty()) {
            System.out.println("Empty order.");
            return;
        }
        orders.add(order);
        System.out.println("Order created #" + order.getId());
    }

    private static void sendToKitchen() {
        for (Order o : orders) if (o.getStatus() == OrderStatus.CREATED) System.out.println(o.shortString());
        int oid = readInt("Enter order id: ", 1, Integer.MAX_VALUE);
        Order o = findOrderById(oid);
        if (o == null || o.getStatus() != OrderStatus.CREATED) {
            System.out.println("Invalid.");
            return;
        }
        KitchenTicket kt = new KitchenTicket(nextTicketId++, o.getId(), o.getItems());
        tickets.add(kt);
        o.setStatus(OrderStatus.SENT_TO_KITCHEN);
        System.out.println("Ticket #" + kt.getId() + " created.");
    }

    private static void generateBill() {
        int oid = readInt("Enter order id: ", 1, Integer.MAX_VALUE);
        Order o = findOrderById(oid);
        if (o == null) return;
        Bill b = new Bill(nextBillId++, o);
        bills.add(b);
        o.setStatus(OrderStatus.BILLED);
        printBill(b);
    }

    private static void recordPayment() {
        for (Bill b : bills) if (!b.isPaid()) System.out.println(b.shortString());
        int bid = readInt("Enter bill id: ", 1, Integer.MAX_VALUE);
        Bill b = findBillById(bid);
        if (b == null || b.isPaid()) return;
        double amt = b.getPayableAmount();
        System.out.println("1. Cash  2. Card");
        int m = readInt("Method: ", 1, 2);
        Payment p;
        if (m == 1) {
            double rec = readDouble("Cash received: ", amt, 10000);
            p = new CashPayment(rec);  // FIXED
        } else {
            System.out.print("Card holder: ");
            String n = sc.nextLine();
            System.out.print("Card no: ");
            String c = sc.nextLine();
            p = new CardPayment(amt, n, c);
        }
        b.recordPayment(p);
        System.out.println("Paid? " + b.isPaid());
        if (!b.getOrder().isTakeaway()) {
            Table t = findTableById(b.getOrder().getTableId());
            if (t != null) t.freeTable();
        }
    }

    private static void displayTablesAndMenu() {
        System.out.println("-- Tables --");
        for (Table t : tables) System.out.println(t);
        System.out.println("-- Menu --");
        displayMenu();
        System.out.println("-- Orders --");
        for (Order o : orders) System.out.println(o.shortString());
        System.out.println("-- Bills --");
        for (Bill b : bills) System.out.println(b.shortString());
    }

    // ========= Helpers =========
    private static void printMainMenu() {
        System.out.println("\n1.Add Table  2.Add Menu Item  3.Seat Customer  4.Create Order");
        System.out.println("5.Send to Kitchen  6.Generate Bill  7.Record Payment  8.Display  9.Exit");
    }
    private static void displayMenu() { for (MenuItem mi : menu) System.out.println(mi); }
    private static void printBill(Bill b) {
        System.out.println("=== Bill #" + b.getId() + " ===");
        for (OrderItem oi : b.getOrder().getItems())
            System.out.println(oi.getQty() + " x " + oi.getMenuItem().getName() + " = " + oi.totalPrice());
        System.out.println("Total: " + b.getPayableAmount());
    }

    private static Table findTableById(int id) { for (Table t : tables) if (t.getId() == id) return t; return null; }
    private static MenuItem findMenuItemById(String id) { for (MenuItem m : menu) if (m.getId().equalsIgnoreCase(id)) return m; return null; }
    private static Order findOrderById(int id) { for (Order o : orders) if (o.getId() == id) return o; return null; }
    private static Bill findBillById(int id) { for (Bill b : bills) if (b.getId() == id) return b; return null; }

    private static int readInt(String msg, int min, int max) {
        while (true) {
            try {
                System.out.print(msg);
                int v = Integer.parseInt(sc.nextLine());
                if (v < min || v > max) continue;
                return v;
            } catch (Exception e) { }
        }
    }
    private static double readDouble(String msg, double min, double max) {
        while (true) {
            try {
                System.out.print(msg);
                double v = Double.parseDouble(sc.nextLine());
                if (v < min || v > max) continue;
                return v;
            } catch (Exception e) { }
        }
    }
    private static void seedSampleData() {
        tables.add(new Table(1,4));
        tables.add(new Table(2,2));
        menu.add(new MenuItem("M1","Pizza",8.5));
        menu.add(new MenuItem("M2","Pasta",6.0));
    }
}

// ============ Domain Classes ============
class Table {
    private final int id,seats;
    private boolean occupied=false;
    private Customer cust;
    public Table(int id,int seats){this.id=id;this.seats=seats;}
    public void seatCustomer(Customer c){cust=c;occupied=true;}
    public void freeTable(){cust=null;occupied=false;}
    public int getId(){return id;}
    public boolean isOccupied(){return occupied;}
    public String toString(){return "Table#"+id+" ("+seats+" seats) - "+(occupied?"Occupied":"Free");}
}
class Customer {
    private final String name;
    public Customer(String n,String c){this.name=n;}
    public String getName(){return name;}
}
class MenuItem {
    private final String id,name; private final double price;
    public MenuItem(String i,String n,double p){id=i;name=n;price=p;}
    public String getId(){return id;} public String getName(){return name;} public double getPrice(){return price;}
    public String toString(){return id+" - "+name+" ($"+price+")";}
}
enum OrderStatus{CREATED,SENT_TO_KITCHEN,BILLED,PAID;}
class Order {
    private final int id,tableId; private final boolean takeaway;
    private final List<OrderItem> items=new ArrayList<>();
    private OrderStatus status=OrderStatus.CREATED;
    public Order(int i,int tid,boolean t){id=i;tableId=tid;takeaway=t;}
    public void addItem(OrderItem oi){items.add(oi);}
    public int getId(){return id;} public int getTableId(){return tableId;}
    public boolean isTakeaway(){return takeaway;}
    public List<OrderItem> getItems(){return items;}
    public OrderStatus getStatus(){return status;} public void setStatus(OrderStatus s){status=s;}
    public String shortString(){return "Order#"+id+" ["+(takeaway?"Takeaway":"Table "+tableId)+"] "+status;}
}
class OrderItem {
    private final MenuItem mi; private final int qty;
    public OrderItem(MenuItem m,int q){mi=m;qty=q;}
    public int getQty(){return qty;} public MenuItem getMenuItem(){return mi;}
    public double totalPrice(){return qty*mi.getPrice();}
}
class KitchenTicket {
    private final int id,orderId; private final List<OrderItem> items;
    public KitchenTicket(int id,int oid,List<OrderItem> it){this.id=id;this.orderId=oid;this.items=new ArrayList<>(it);}
    public int getId(){return id;} public int getOrderId(){return orderId;} public List<OrderItem> getItems(){return items;}
}
class Bill {
    private final int id; private final Order order; private boolean paid=false; private Payment pay;
    public Bill(int i,Order o){id=i;order=o;}
    public int getId(){return id;} public Order getOrder(){return order;}
    public double getPayableAmount(){double s=0;for(OrderItem oi:order.getItems())s+=oi.totalPrice();return s*1.15;}
    public void recordPayment(Payment p){if(p.process(getPayableAmount())){paid=true;pay=p;order.setStatus(OrderStatus.PAID);}}
    public boolean isPaid(){return paid;}
    public String shortString(){return "Bill#"+id+" Order#"+order.getId()+" Pay:"+getPayableAmount()+" Paid:"+paid;}
}
abstract class Payment {
    protected final double amount;
    public Payment(double a){amount=a;}
    public abstract boolean process(double expected);
}
class CashPayment extends Payment {
    private final double received;
    public CashPayment(double r){super(r);this.received=r;}
    public boolean process(double exp){return received>=exp;}
}
class CardPayment extends Payment {
    private final String holder,card;
    public CardPayment(double a,String h,String c){super(a);holder=h;card=c;}
    public boolean process(double exp){return amount==exp;}
}

