# Extending

### Events

The export process has a few events you can leverage to interact with the underlying 
classes to add custom behaviour to the export.

You are able to hook into the parent package by using events.
No need to use convenience methods like "query" or "view", if you need full control over the export.

The events will be activated by adding the `WithEvents` concern. Inside the `registerEvents` method, you 
will have to return an array of events. The key is the Fully Qualified Name (FQN) of the event and the value is a callable event listener.
This can either be a closure, array-callable or invokable class.

```php
namespace App\Exports;

use Maatwebsite\Excel\Concerns\WithEvents;
use Maatwebsite\Excel\Events\BeforeExport;
use Maatwebsite\Excel\Events\BeforeWriting;
use Maatwebsite\Excel\Events\BeforeSheet;

class InvoicesExport implements WithEvents
{
    /**
     * @return array
     */
    public function registerEvents(): array
    {
        return [
            // Handle by a closure.
            BeforeExport::class => function(BeforeExport $event) {
                $event->writer->getProperties()->setCreator('Patrick');
            },
            
            // Array callable, refering to a static method.
            BeforeWriting::class => [self::class, 'beforeWriting'],
            
            // Using a class with an __invoke method.
            BeforeSheet::class => new BeforeSheetHandler()
        ];
    }
    
    public static function beforeWriting(BeforeWriting $event) 
    {
        //
    }
}
```

Do note that using a `Closure` will not be possible in combination with queued exports, as PHP cannot serialize the closure.
In those cases it might be better to use the `RegistersEventListeners` trait.

#### Auto register event listeners

By using the `RegistersEventListeners` trait you can auto-register the event listeners,
without the need of using the `registerEvents`. The listener will only be registered if the method is created. 

```php
namespace App\Exports;

use Maatwebsite\Excel\Concerns\WithEvents;
use Maatwebsite\Excel\Concerns\RegistersEventListeners;
use Maatwebsite\Excel\Events\BeforeExport;
use Maatwebsite\Excel\Events\BeforeWriting;
use Maatwebsite\Excel\Events\BeforeSheet;
use Maatwebsite\Excel\Events\AfterSheet;

class InvoicesExport implements WithEvents
{
    use Exportable, RegistersEventListeners;
    
    public static function beforeExport(BeforeExport $event)
    {
        //
    }

    public static function beforeWriting(BeforeWriting $event)
    {
        //
    }

    public static function beforeSheet(BeforeSheet $event)
    {
        //
    }

    public static function afterSheet(AfterSheet $event)
    {
        //
    }
}
```

### Global event listeners

Event listeners can also be configured globally, if you want to perform the same actions on all exports in your app.
You can add them to e.g. your `AppServiceProvider` in the `register()` method.

```php
Writer::listen(BeforeExport::class, function () {
    //
});

Writer::listen(BeforeWriting::class, function () {
    //
});

Sheet::listen(BeforeSheet::class, function () {
    //
});

Sheet::listen(AfterSheet::class, function () {
    //
});
```

#### Available events

| Event name | Payload | Explanation |
|---- |----| ----|
|`Maatwebsite\Excel\Events\BeforeExport` | `$event->writer : Writer` | Event gets raised at the start of the process. | 
| `Maatwebsite\Excel\Events\BeforeWriting` | `$event->writer : Writer` | Event gets raised before the download/store starts. |
| `Maatwebsite\Excel\Events\BeforeSheet` | `$event->sheet : Sheet` | Event gets raised just after the sheet is created. |
| `Maatwebsite\Excel\Events\AfterSheet` | `$event->sheet : Sheet` | Event gets raised at the end of the sheet process. |


### Macroable

Both `Writer` and `Sheet` are "macroable" which means they can easily be extended to fit your needs. 
Both Writer and Sheet have a `->getDelegate()` method which returns the underlying PhpSpreadsheet class. 
This will allow you to add custom macros as shortcuts to PhpSpreadsheet methods that are not available in this package. 



#### Writer

```php
use \Maatwebsite\Excel\Writer;

Writer::macro('setCreator', function (Writer $writer, string $creator) {
    $writer->getDelegate()->getProperties()->setCreator($creator);
});
```

#### Sheet

```php
use \Maatwebsite\Excel\Sheet;

Sheet::macro('setOrientation', function (Sheet $sheet, $orientation) {
    $sheet->getDelegate()->getPageSetup()->setOrientation($orientation);
});
```

#### Customize  
You can create your own macro to add custom methods to a spreadsheet instance. 

For example, to add styling to a cell, we first create a macro, let's call it `styleCells`:


```php
use \Maatwebsite\Excel\Sheet;

Sheet::macro('styleCells', function (Sheet $sheet, string $cellRange, array $style) {
    $sheet->getDelegate()->getStyle($cellRange)->applyFromArray($style);
});
```

Once the macro is created, it can be used inside the `registerEvents()` method as such:

```php
namespace App\Exports;

use Maatwebsite\Excel\Concerns\WithEvents;
use Maatwebsite\Excel\Events\BeforeExport;
use Maatwebsite\Excel\Events\AfterSheet;

class InvoicesExport implements WithEvents
{
    /**
     * @return array
     */
    public function registerEvents(): array
    {
        return [
            BeforeExport::class  => function(BeforeExport $event) {
                $event->writer->setCreator('Patrick');
            },
            AfterSheet::class    => function(AfterSheet $event) {
                $event->sheet->setOrientation(\PhpOffice\PhpSpreadsheet\Worksheet\PageSetup::ORIENTATION_LANDSCAPE);

                $event->sheet->styleCells(
                    'B2:G8',
                    [
                        'borders' => [
                            'outline' => [
                                'borderStyle' => \PhpOffice\PhpSpreadsheet\Style\Border::BORDER_THICK,
                                'color' => ['argb' => 'FFFF0000'],
                            ],
                        ]
                    ]
                );
            },
        ];
    }
}
```

Feel free to use the above macro, or be creative and invent your own!

For PhpSpreadsheet methods, please refer to their documentation: https://phpspreadsheet.readthedocs.io/