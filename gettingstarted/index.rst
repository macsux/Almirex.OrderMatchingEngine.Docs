Getting Started
===============
The order matching engine is based around 4 main components:

* ``ExchangeEngine`` - a collection of orderbooks identified by unique symbols. Orderbooks must be explicitly created by calling ``CreateMarket`` method. Orders can be submitted directly to the ``ExchangeEngine`` and will be added to delegated to the appropriate orderbook for execution, or rejected if market is not created.
* ``OrderBook`` - a marketplace serving a single type of product being traded identified by symbol. OrderBook can be used on their own or as part of ``ExchangeEngine`` class. Any orders added to the OrderBook will automatically follow standard rules which will cause events to fire. 
* ``Order`` - the core component that defines the properties of the order. Orders are mutable in nature and their field values will change according to standard exchange rules
* ``ExecutionReport`` - represents an event that have occured to the order and captures a snapshot of the order and all fields related to the event type. The event type can be determined by looking at the ``ExecType`` field.

Creating orders
---------------
The following example adds a single order to the orderbook. This causes a single execution report to be created with status of New, representing a successful addition to the orderbook

.. sourcecode:: csharp

      var engine = new ExchangeEngine("my engine", ExchangeEngineOptions.Default);
      var orderBook = engine.CreateMarket("USD/CAD");
      var order = new Order
      {
          Symbol = "USD/CAD",
          Side = Side.Buy,
          ClientID = "client1",
          OrdType = OrdType.Limit,
          OrderQty = 10,
          Price = 30
      };
      engine.NewOrder(order);
    
Trading
-------
Trading is performed automatically when an action performed against an orderbook (either via adding, or causing an order to trigger) which results in a negative spread. The engine will continue trading orders until spread becomes positive or one side of the orderbook becomes empty.  Assuming we have prepared the following scenario:

.. sourcecode:: csharp

    string symbol = "USD/CAD";
    var orderBook = new OrderBook(symbol);
    var orderSell = new Order
    {
        Symbol = symbol,
        Side = Side.Sell,
        ClientID = "client1",
        OrdType = OrdType.Limit,
        OrderQty = 5,
        Price = 25
    };
    var orderBuy = new Order
    {
        Symbol = symbol,
        Side = Side.Buy,
        ClientID = "client2",
        OrdType = OrdType.Limit,
        OrderQty = 10,
        Price = 30
    };

Trades can be captured in one of two ways:

Option 1 - Subscribing to ``Trade`` event
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
.. sourcecode:: csharp

    orderBook.Trade += (sender, args) =>
    {
        Console.WriteLine($"Traded {args.Amount} {args.TakingOrder.Symbol} at ${args.Price}");
    };
    orderBook.NewOrder(orderBuy);
    orderBook.NewOrder(orderSell); // <-- event will be fired on this line
            
Option 2 - Capturing the trades into ``ExecutionReport``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
.. sourcecode:: csharp

    // two execution reports will be produced, one for each side of the order. 
    // the two parts can be correlated via common id in TradeId field
    orderBook.TradeIdGenerator = () => Guid.NewGuid().ToString(); 
    var executionReports = orderBook.WithReports(ob =>
    {
        ob.NewOrder(orderBuy);
        ob.NewOrder(orderSell);
    });
    var tradeReportOrder1 = executionReports[2];
    var tradeReportOrder2 = executionReports[3];
    Console.WriteLine($"Traded {tradeReportOrder1.LastQty} {tradeReportOrder1.Symbol} at ${tradeReportOrder1.LastPx}");
    Console.WriteLine($"TradeID: {tradeReportOrder1.TradeId} == {tradeReportOrder2.TradeId}");
