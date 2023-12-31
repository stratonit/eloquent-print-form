# Print forms for Eloquent

Простой способ генерации форматов печати docx для счетов, контрактов и других документов.

## Использование

Установка

```bash
composer require stratonit/eloquent-print-form
```

Объявите модели для Eloquent.

Создайте форму для печати (файл docx), используя следующий синтаксис.

- `${field_name}` - стандартное поле
- `${entity.field_name}` - поле из связанной сущности
- `${entity.field_name|date}` - форматировать поле как дату
- `${entity.field_name|date|placeholder}` - форматировать поле как дату и заменить на "____________________", если поле пустое
- `${entity1.entity2.field_name}` - поле из связанной сущности через одну таблицу
- `${entities.field_name}` - для данных таблицы
Как вы можете видеть, возможно использование "труб" (pipes), например: |date, |date|placeholder. Поддерживаемые "трубы":

- `placeholder` - заменяет пустую переменную на "____________________".
- `date` - форматирует дату
- `dateTime` - форматирует дату и время
- `int` - форматирует целое число
- `decimal` - форматирует десятичное число
Также возможно указание пользовательских "труб".

Сгенерировать форму для печати для указанной сущности.

```php
$entity = Contract::find($id);
$printFormProcessor = new PrintFormProcessor();
$templateFile = resource_path('path_to_print_forms/your_print_form.docx');
$tempFileName = $printFormProcessor->process($templateFile, $entity);
```

## Примеры

### Базовый пример

Допустим, в проекте есть следующие модели.

```php
use Illuminate\Database\Eloquent\Model;

/**
 * @property string $number
 * @property string $start_at
 * @property int $customer_id
 * @property Customer $customer
 * @property ContractAppendix[] $appendixes
 */
class Contract extends Model
{

    public function customer()
    {
        return $this->belongsTo(Customer::class);
    }
 
    public function appendixes()
    {
        return $this->hasMany(ContractAppendix::class);
    }
}

/**
 * @property string $name
 * @property CustomerCategory $category
 */
class Customer extends Model
{
    public function category()
    {
        return $this->belongsTo(CustomerCategory::class);
    }

}

/**
 * @property string $number
 * @property string $date
 * @property float $tax
 */
class ContractAppendix extends Model
{

    public function items()
    {
        return $this->hasMany(ContractAppendixItem::class, 'appendix_id');
    }

    public function getTaxAttribute()
    {
        $tax = 0;
        foreach ($this->items as $item) {
            $tax += $item->total_amount * (1 - 100 / (100+($item->taxStatus->vat_rate ?? 0)));
        }
        return $tax;
    }
}
```

Пример действия контроллера в Laravel для печати контракта

```php
public function downloadPrintForm(FormRequest $request)
{
    $id = $request->get('id');
    $entity = Contract::find($id);
    $printFormProcessor = new PrintFormProcessor();
    $templateFile = resource_path('path_to_print_forms/your_print_form.docx');
    $tempFileName = $printFormProcessor->process($templateFile, $entity);
    $filename = 'contract_' . $id;
    return response()
        ->download($tempFileName, $filename . '.docx')
        ->deleteFileAfterSend();
}
```

Пример шаблона docx

---

- Номер контракта: `${number|placeholder}`
- Дата контракта: `${start_at|date|placeholder}`
- Клиент: `${customer.name}`
- Категория клиента: `${customer.category.name|placeholder}`

Приложения:


| Row number                | Appendix number      | Appendix tax      |
| ------------------------- | -------------------- | -----------------:|
| ${appendixes.#row_number} | ${appendixes.number} | ${appendixes.tax} |

---

Пример сгенерированного документа

---

- Номер контракта: `F-123`
- Дата контракта: `12.04.2012`
- Клиент: `IBM`
- Категория клиента: `____________________`

Приложения:
| Row number                | Appendix number      | Appendix tax      |
| ------------------------- | -------------------- | -----------------:|
| 1                         | A-1                  | 1234              |
| 2                         | A-1.1                | 0                 |
| 3                         | B-1                  | 10                |

---

### Как указать пользовательские "конвеер" (pipes)

```php
class CustomPipes extends \Mnvx\EloquentPrintForm\Pipes
{
    // Override
    public function placeholder($value)
    {
        return $value ?: "NO DATA";
    }

    // New pipe
    public function custom($value)
    {
        return "CUSTOM";
    }
}

$entity = Contract::find($id);
$pipes = new CustomPipes();
$printFormProcessor = new PrintFormProcessor($pipes);
$templateFile = resource_path('path_to_print_forms/your_print_form.docx');
$tempFileName = $printFormProcessor->process($templateFile, $entity);
```

### Как указать пользовательские обработчики для некоторых переменных

Пример пользовательского обработчика для переменной `${custom}`.

```php
$tempFileName = $printFormProcessor->process($templateFile, $entity, [
    'custom' => function(TemplateProcessor $processor, string $variable, ?string $value) {
        $inline = new TextRun();
        $inline->addText('by a red italic text', array('italic' => true, 'color' => 'red'));
        $processor->setComplexValue($variable, $inline);
    },
]);
```
