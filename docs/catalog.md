# Работа с каталогом

## ТОРГОВЫЕ ПРЕДЛОЖЕНИЯ
```php
\Bitrix\Main\Loader::includeModule('catalog');

//Товары связаны в торговыми предложениями через свойство CML2_LINK инфоблока торговых предложений. Тут пишется id товара.
$elementObjects = \Bitrix\Iblock\Elements\ElementOfferOilEntityTable::query()
    ->registerRuntimeField(
        null, //всегда null
        //Присодинять нужно именно таблицу каталога к таблице торговых предложений, иначе ошибка будет из-за того,
        // что в join для ref нельзя указывать вложенные поля типа ref.CML2_LINK.VALUE
        (new Reference(
            'REF_SKU',
            \Bitrix\Iblock\Elements\ElementCatalogOilEntityTable::class,
            Join::on('this.CML2_LINK.VALUE', 'ref.ID'),
        ))->configureJoinType(Join::TYPE_INNER)
    )
    ->addSelect('ID')
    ->addSelect('NAME')
    ->where('REF_SKU.ID', 3313)
    ->fetchCollection();
$offers         = [];
foreach ($elementObjects as $elementObject) {
    $offers[] = [
        'id'   => $elementObject->getId(),
        'name' => $elementObject->getName(),
    ];
}
```
## ОСТАТКИ И ПАРАМЕТРЫ ТОВАРА

Данные, которые указываются на вкладке "Торговый каталог" -> "Параметры".
Если товар - "Простой", то нужно указывать его id, а если это "Товар с торговыми предложениями", то надо указывать id торгового предложения.

Описание полей можно глянуть тут - https://dev.1c-bitrix.ru/api_help/catalog/classes/ccatalogproduct/ccatalogproduct__getlist.971a2b70.php
Дока для устаревшего класса, но поля в основном те же самые

```php

$productObjects = \Bitrix\Catalog\ProductTable::query()
    ->addSelect('QUANTITY') //Количество
    ->addSelect('QUANTITY_RESERVED') //Зарезервированное количество
    ->addSelect('WEIGHT') //Вес в граммах
    ->addSelect('WIDTH') //Ширина в мм
    ->addSelect('LENGTH') //Длина в мм
    ->addSelect('HEIGHT') //Высота в ММ
    //Тип товара - одна из нижеуказанных констант
    //\Bitrix\Catalog\ProductTable::TYPE_PRODUCT = 1; - простой продукт
    //\Bitrix\Catalog\ProductTable::TYPE_SET = 2; - набор
    //\Bitrix\Catalog\ProductTable::TYPE_SKU = 3; - товар с торговыми предложениями
    //\Bitrix\Catalog\ProductTable::TYPE_OFFER = 4; - торговое предложение
    //\Bitrix\Catalog\ProductTable::TYPE_FREE_OFFER = 5; - торговое предложение без привязки к товару (обычно такого быть не должно)
    //\Bitrix\Catalog\ProductTable::TYPE_EMPTY_SKU = 6;
    //\Bitrix\Catalog\ProductTable::TYPE_SERVICE = 7;
    ->addSelect('TYPE')
    //Y/N, доступен ли товар к покупке - рассчитываемое значение, зависит от настроек (количественный учет включен или нет, разрешена ли покупка при 0 и т.д.)
    ->addSelect('AVAILABLE')
    ->addSelect('CAN_BUY_ZERO') //можно ли покупать товар при отсутствии остатков (остатки просто будут уходить в минус)
    ->addSelect('QUANTITY_TRACE') //Y/N - ведется ли количественный учет по товару
    ->where('ID', 3314)
    ->fetchCollection();
$products       = [];
foreach ($productObjects as $productObject) {
    $products[] = [
        'quantity'         => $productObject->getQuantity(),
        'quantityReserved' => $productObject->getQuantityReserved(),
        'weightInGrams'    => $productObject->getWeight(),
        'width'            => $productObject->getWidth(),
        'length'           => $productObject->getLength(),
        'height'           => $productObject->getHeight(),
        'type'             => $productObject->getType(),
        'available'        => $productObject->getAvailable(),
    ];
}
```
## ЦЕНА, СКИДКИ
```php
//Выбирает самую низкую цену (если товару задано несколько типов цен) для указанных групп пользователя и применяет скидки
$productId       = 3314; //id товара (если товар простой) или id торгового предложения
$allUsersGroupId = 2;
$optimalPrice    = \CCatalogProduct::GetOptimalPrice($productId, [$allUsersGroupId]);
```
## СКЛАДЫ
Склады в битрикс, если НЕ включен Складской учет, несут справочную функцию, при оформлении заказа не происходит списание со склада.
Складской учет включается достаточно редко и его нельзя включать, если настроена интеграция с 1С.
```php
// Получаем остатки товара
$productId          = 3314;
$storeProductObject = \Bitrix\Catalog\StoreProductTable::query()
    ->addSelect('STORE_ID')
    ->addSelect('AMOUNT')
    ->where('PRODUCT_ID', $productId)
    ->fetchCollection();

$stocks = [];
foreach ($storeProductObject as $storeProductObject) {
    $stocks[$storeProductObject->getStoreId()] = $storeProductObject->getAmount();
}
```