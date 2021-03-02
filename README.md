# MyShop-UnitOfWork

1. Unit of Work in C# is the concept that is related to the effective implementation of the Repository Design Pattern.
2. So, to understand this concept it is important to understand the concept of the Repository Pattern.
3. The Unit of Work pattern is used to group one or more operations (usually database CRUD operations) into a single transaction or “unit of work” so that all operations either pass or fail as one.
4.Things get complicated when a set of database commit operations on one entity depend on a commit operation on another entity.
5.instead we can further improve the design and efficiency of the data transactions across the application repository layers by clustering related repository components into a single unit, called as a **UnitOfWork**.
6.so we can reduce the number of times a database connection is made for transaction.

## let's see a use case
1. Customer can create an order.
2. If the Customer already has an account, we  update his information and save his new information in the database (**we make a Transaction to the database**).
3. If not, we create a New Customer.
4. Then, make an orderObject and save it in the database (**we make another Transaction to the database**)
5. 
### so, what is the problem here?
1. We make two transactions for the same operation.
2. Both repositories[CustomerRepositore && OrderRepository] will generate and maintain their own instance of the DbContext class.
3. If the SaveChanges of one of them fails and the other one succeeds, it will result in database inconsistency.

```c#
[HttpPost]
        public IActionResult Create(CreateOrderModel model)
        {
            if (!model.LineItems.Any()) return BadRequest("Please submit line items");

            if (string.IsNullOrWhiteSpace(model.Customer.Name)) return BadRequest("Customer needs a name");

            var customer = customerRepository.Find(c => c.Name == model.Customer.Name)
                          .FirstOrDefault();
            if (customer != null)
            {
                customer.ShippingAddress = model.Customer.ShippingAddress;
                customer.PostalCode = model.Customer.PostalCode;
                customer.City = model.Customer.City;
                customer.Country = model.Customer.Country;
                customerRepository.Update(customer);
                customerRepository.SaveChanges();

            }
            else {
                 customer = new Customer
                {
                    Name = model.Customer.Name,
                    ShippingAddress = model.Customer.ShippingAddress,
                    City = model.Customer.City,
                    PostalCode = model.Customer.PostalCode,
                    Country = model.Customer.Country
                };
            }
           

            var order = new Order
            {
                LineItems = model.LineItems
                    .Select(line => new LineItem { ProductId = line.ProductId, Quantity = line.Quantity })
                    .ToList(),

                Customer = customer
            };

            orderRepository.Add(order);

            orderRepository.SaveChanges();

            return Ok("Order Created");
        }
```

## implementation 
### 1-create IUnitOfWork interface
### 2-create UnitOfWork class
```c#
public interface IUnitOfWork
    {
        //One should only group all the Repositories representing entities which are related to one another and depend on one another into a single UnitOfWork. 
        //Unless an application contains such a scenario, UnitOfWork isn't required to be implemented.
        
        
        IRepository<Customer> CustomerRepository { get; }
        IRepository<Order> OrderRepository { get; }
        IRepository<Product> ProductRepository { get; }

        void SaveChanges();
    }

    public class UnitOfWork : IUnitOfWork
    {
        //all  Repository classes receive the same context object reference on which the single Save() method works on.
        private ShoppingContext context;

        public UnitOfWork(ShoppingContext context)
        {
            this.context = context;
        }

        private IRepository<Customer> customerRepository;
        public IRepository<Customer> CustomerRepository
        {
            get
            {
                if (customerRepository == null)
                {
                    customerRepository = new CustomerRepository(context);
                }

                return customerRepository;
            }
        }

        private IRepository<Order> orderRepository;
        public IRepository<Order> OrderRepository
        {
            get
            {
                if(orderRepository == null)
                {
                    orderRepository = new OrderRepository(context);
                }

                return orderRepository;
            }
        }

        private IRepository<Product> productRepository;
        public IRepository<Product> ProductRepository
        {
            get
            {
                if (productRepository == null)
                {
                    productRepository = new ProductRepository(context);
                }

                return productRepository;
            }
        }

        public void SaveChanges()
        {
            context.SaveChanges();
        }
    }
```
### 3-Use it in Controller class
```c#
       public class OrderController : Controller {
       
        private readonly IUnitOfWork unitOfWork;

        public OrderController(IUnitOfWork unitOfWork)
        {
        
            this.unitOfWork = unitOfWork;
        }

        public IActionResult Index()
        {
            var orders = unitOfWork.OrderRepository.Find(order => order.OrderDate > DateTime.UtcNow.AddDays(-1));

            return View(orders);
        }

        public IActionResult Create()
        {
            var products = unitOfWork.ProductRepository.All();

            return View(products);
        }

        [HttpPost]
        public IActionResult Create(CreateOrderModel model)
        {
            if (!model.LineItems.Any()) return BadRequest("Please submit line items");

            if (string.IsNullOrWhiteSpace(model.Customer.Name)) return BadRequest("Customer needs a name");

            var customer = unitOfWork.CustomerRepository
                .Find(c => c.Name == model.Customer.Name)
                .FirstOrDefault();

            if(customer != null)
            {
                customer.ShippingAddress = model.Customer.ShippingAddress;
                customer.PostalCode = model.Customer.PostalCode;
                customer.City = model.Customer.City;
                customer.Country = model.Customer.Country;

                unitOfWork.CustomerRepository.Update(customer);
            }
            else
            {
                customer = new Customer
                {
                    Name = model.Customer.Name,
                    ShippingAddress = model.Customer.ShippingAddress,
                    City = model.Customer.City,
                    PostalCode = model.Customer.PostalCode,
                    Country = model.Customer.Country
                };
            }

            var order = new Order
            {
                LineItems = model.LineItems
                    .Select(line => new LineItem { ProductId = line.ProductId, Quantity = line.Quantity })
                    .ToList(),

                Customer = customer
            };

            unitOfWork.OrderRepository.Add(order);

            unitOfWork.SaveChanges();

            return Ok("Order Created");
        }

        [ResponseCache(Duration = 0, Location = ResponseCacheLocation.None, NoStore = true)]
        public IActionResult Error()
        {
            return View(new ErrorViewModel { RequestId = Activity.Current?.Id ?? HttpContext.TraceIdentifier });
        }
    }
    
    //in startup
        public void ConfigureServices(IServiceCollection services)
        {
            //Di 
            services.AddTransient<IUnitOfWork, UnitOfWork>();
        }
        
```
