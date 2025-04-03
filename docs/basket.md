# Работа с корзиной

FUserId - уникальный идентификатор, который есть у каждого пользователя, в т.ч. неавторизованного. Служит для привязки к нему корзины.
Хранится в таблице b_sale_fuser
Корзина - список товаров, который хранится в таблице b_sale_basket. Каждый товар привязан к FUSER_ID. Если оформлен заказ, то еще и
проставлено поле привязки к заказу ORDER_ID. При оформлении заказа корзина никуда не перемещается, просто проставляестя привязка к заказу.

## ЧТЕНИЕ ДАННЫХ
### ЧТЕНИЕ ДАННЫХ ПОСРЕДСТВОМ СПЕЦИАЛЬНЫХ ОБЪЕКТОВ
Это полезно, если вам надо получить какие-либо общие данные по корзине, например, получить сумму корзины со скидкой.
Также если вы планируете менять какие-то данные в корзине - в этом случае нужно работать исключительно через объект \Bitrix\Sale\Basket, чтобы обеспечить целостность данных корзины.
```php
\Bitrix\Main\Loader::includeModule('sale');

//загрузить корзину для текущего пользователя (в т.ч. неавторизованного)
$basket = \Bitrix\Sale\Basket::loadItemsForFUser(\Bitrix\Sale\Fuser::getId(), 's1');

//загрузить корзину по заказу
$order  = \Bitrix\Sale\Order::load(10944);
$basket = \Bitrix\Sale\Basket::loadItemsForOrder($order);
//или
$basket = $order->getBasket();

//Информация о корзине
$price         = $basket->getPrice(); // Цена с учетом скидок
$fullPrice     = $basket->getBasePrice(); // Цена без учета скидок
$weight        = $basket->getWeight(); // Общий вес корзины
$basketFUserId = $basket->getFUserId(); //fUserId, к которому привязана корзина
$basketOrder   = $basket->getOrder(); //объект \Bitrix\Sale\Order или null, если корзина не привязана заказу
$basketSiteId  = $basket->getSiteId(); //id сайта, к которому привязана корзина

//Товары в корзине
$basketItems = $basket->getBasketItems();
//получить товары, доступные к заказу. Например, те, которые есть в наличии, если не разрешено продавать товары не в наличии
$basketItems = $basket->getOrderableItems();

//Перебор товаров
/** @var \Bitrix\Sale\BasketItem $basketItem */
foreach ($basketItems as $basketItem) {
    $basketItem->getId();         // ID записи в корзине
    $basketItem->getProductId();  // ID товара
    $basketItem->getPrice();      // Цена за единицу со скидками
    $basketItem->getBasePrice();  // Цена за единицу без скидок
    $basketItem->getQuantity();   // Количество
    $basketItem->getFinalPrice(); // Сумма
    $basketItem->getWeight();     // Вес в граммах
    $basketItem->getField('NAME');// Любое поле товара в корзине
    $basketItem->canBuy();
    $basketItem->isDelay();       // true, если отложено

    //Получить коллекцию свойств товара в корзине. Cвойства товара в корзине не то же самое, что свойства товара в инофблоке. Это набор произвольных атрибутов. Обычно сюда копируются некоторые свойства инфоблока,
    // но можно добавить и произвольное свойство при добавлении товара в корзину
    /** @var \Bitrix\Sale\BasketPropertiesCollection $itemProperties */
    $itemProperties = $basketItem->getPropertyCollection();
    //получить массив свойств
    $itemPropertyValues = $itemProperties->getPropertyValues();

    //Пример удаления свойства товара
    foreach ($itemProperties as $propertyItem) {
        if ($propertyItem->getField('CODE') == 'COLOR') {
            $propertyItem->delete();
            break;
        }
    }
    $itemProperties->save();

    //добавить новое свойство товара
    $itemProperties->createItem()
        ->setFields([
            'NAME'  => 'Id дилера',
            'CODE'  => 'DEALER_ID',
            'VALUE' => 2180,
        ]);
    $itemProperties->save();
}
```
### ЧТЕНИЕ ДАННЫХ С ПОМОЩЬЮ ORM-КЛАССОВ
Это может быть полезно, если вам надо получить данные исключительно для чтения, например, список товаров корзины. Запрос через ORM - более легкий и удобный способ.
```php
$itemObjects = \Bitrix\Sale\Internals\BasketTable::query()
    ->addSelect('BASKET_COUNT')
    ->addSelect('BASKET_SUM')
    ->registerRuntimeField(
        null,
        new \Bitrix\Main\Entity\ExpressionField('BASKET_COUNT', 'COUNT(*)')
    )
    ->registerRuntimeField(
        null,
        new \Bitrix\Main\Entity\ExpressionField('BASKET_SUM', 'SUM(PRICE*QUANTITY)')
    )
    ->where('FUSER_ID', \Bitrix\Sale\Fuser::getId())
    ->whereNull('ORDER_ID')
    ->where('LID', SITE_ID)
    ->where('CAN_BUY', 'Y')
    ->fetchCollection();
foreach ($itemObjects as $itemObject) {
    $countItems = $itemObject->get('BASKET_COUNT');
}
```
## ДОБАВЛЕНИЕ ТОВАРА В КОРЗИНУ
```php
\Bitrix\Main\Loader::includeModule('catalog');
$productId        = 105;
$fields           = [
    'PRODUCT_ID' => $productId, // ID товара, обязательно
    'QUANTITY'   => 2, // количество, обязательно
    'PROPS'      => [ //свойства товара - необязательный параметр
        [
            'FASOVKA'                    => 'бочка',
            'PRODUCER'                   => 'Хендэ',
            'CHTO_XOCHU_TO_I_DOBAVLYAYU' => 'mogu sebe pozvolit',
        ],
    ],
];
$addProductResult = Bitrix\Catalog\Product\Basket::addProduct($fields);
if (!$addProductResult->isSuccess()) {
    throw new \RuntimeException('Ошибки при добавлении в корзину товара ID ' . $productId . ': ' . implode(', ', $addProductResult->getErrorMessages());
}
```
## УДАЛЕНИЕ ТОВАРА ИЗ КОРЗИНЫ
```php
//1. Перебор товаров в корзине
$basketItems = $basket->getBasketItems();
/** @var \Bitrix\Sale\BasketItem $basketItem */
foreach ($basketItems as $basketItem) {
    if ($basketItem->getProductId() === $productId) {
        $basketItem->delete();
    }
}
$basket->save();

//2. Получение по id (обратите внимание - это не id товара, а id записи в таблице b_sale_basket
$id = 231234;
$basket->getItemById($id)->delete();
$basket->save();
```
## РАССЧЕТ СКИДОК
Скидки по умолчанию не будут рассчитаны в объекте \Bitrix\Sale\Basket, который не привязан к заказу
Код ниже написан по мотивам компонента bitrix:sale.basket.basket - он привязывает объект заказа (несохраненный в БД) к корзине и так рассчитывает все скидки
```php
$registry = \Bitrix\Sale\Registry::getInstance(\Bitrix\Sale\Registry::REGISTRY_TYPE_ORDER);
/** @var Order $orderClass */
$orderClass = $registry->getOrderClassName();
$userId = 2312;
$tempOrder = $orderClass::create($basket->getSiteId(), $userId);

$result = $tempOrder->appendBasket($basket);
if (!$result->isSuccess()) {
    throw new RuntimeException('Ошибки при инициализации объекта заказа для корзины: ' . implode(', ', $result->getErrors()), 400);
}

$discounts  = $tempOrder->getDiscount();
$showPrices = $discounts->getShowPrices();
if (empty($showPrices['BASKET'])) {
    return;
}
foreach ($showPrices['BASKET'] as $basketCode => $data) {
    $basketItem = $basket->getItemByBasketCode($basketCode);
    if (!($basketItem instanceof BasketItemBase)) {
        continue;
    }
    $basketItem->setFieldNoDemand('BASE_PRICE', $data['SHOW_BASE_PRICE']);
    $basketItem->setFieldNoDemand('PRICE', $data['SHOW_PRICE']);
    $basketItem->setFieldNoDemand('DISCOUNT_PRICE', $data['SHOW_DISCOUNT']);
}
//тут будет список примененных скидок
$discountData = $discounts->getApplyResult(true);
```