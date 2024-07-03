That is how this screen is built. You can use custom pages, clusters, and  getRecordSubNavigation to achieve this.
This is how I built this:
1. Create custom pages for each page in the navigation cluster
```php artisan make:filament-page OrderSummary --resource=OrderResource --type=custom
  ...
  php artisan make:filament-page OrderCriteria --resource=OrderResource --type=custom
```
2. Add those pages to the the getPages() function in OrderResource
 ```
 'edit' => OrderSummary::route('/{record}/summary'),
            'rental-history' => OrderRentalHistory::route('/{record}/rental-history'),
            'income' => OrderIncome::route('/{record}/income'),
            'credit' => OrderCredit::route('/{record}/credit'),
            'crime' => OrderCrime::route('/{record}/crime'),
            'evictions' => OrderEviction::route('/{record}/evictions'),
            'application' => OrderApplication::route('/{record}/application'),
            'invoice' => OrderInvoice::route('/{record}/invoice'),
            'files' => OrderFiles::route('/{record}/files'),
            'audit-log' => OrderAuditLog::route('/{record}/audit-log'),
            'criteria' => OrderCriteria::route('/{record}/criteria'),
```
3. Add getRecordSubNavigation to OrderResource
```
 public static function getRecordSubNavigation(Page $page): array
    {
        return $page->generateNavigationItems([
            OrderSummary::class,
            OrderRentalHistory::class,
            OrderIncome::class,
            OrderCredit::class,
            OrderCrime::class,
            OrderEviction::class,
            OrderApplication::class,
            OrderInvoice::class,
            OrderFiles::class,
            OrderAuditLog::class,
            OrderCriteria::class,
        ]);
    }
```
4. Create a cluster
```
   php artisan make:filament-cluster OrderCluster
```
5.  Add the cluster to each custom page created in step 1
   (e.g. file: 'OrderSummary')
```
    protected static ?string $cluster = OrderCluster::class;
    protected static string $resource = OrderResource::class;

```
6. Set how you want subnavigation to appear in your resource
(OrderResource.php)
```
// Use SubNavigationPosition::top to create tabs across the top
 protected static SubNavigationPosition $subNavigationPosition = SubNavigationPosition::Start;4
```
7. Create a common trait that can be included in each custom page (Multi-section form)

I called this trait (InOrderCluster.php)
```
 abstract protected function getPageFormSections(): array;
 abstract protected function loadPageRelations(): void;
 abstract protected function getPageRouteName(): string;

 public ?array $data = [];

 public function getView(): string
    {
       // Common View for all multi-section form pages
        return 'modules.order.filament.clusters.order.resources.order-resource.pages.order-cluster';
    }
 
    protected function mountOrderCluster(): void
    {

        // Load the relationships needed for the cluster common order details
        $this->loadOrderDetails();
        // Load specific relations for the parent page, abstract function requires implementation
        $this->loadPageRelations();
    
        $this->fillForm();
    }

    public function fillForm(): void
    {
        $data = $this->record->toArray();

        $this->form->fill($data);
    }

    public function form(Form $form): Form
    {
        return $form->schema([
            Grid::make()->schema([
                Grid::make()->schema(
                    // main forms w/in this section, abstract requires implementation
                    $this->getPageFormSections()
                )->columnSpan(4),
                OrderCluster::getOrderDetails(columnSpan: 2)->columnSpan(2),
            ])->columns(6),

        ])
            ->model($this->record)
            ->statePath('data')
            ->operation('edit');
    }

```
8. Create a common view page for all multi-section form pages
(order-cluster.blade.php)
```
<x-filament-panels::page>
    <x-filament-panels::form>
        {{ $this->form }}

    </x-filament-panels::form>
    <x-filament-actions::modals/>
</x-filament-panels::page>

```
9. Add the common trait to all custom multi-section pages
```
  use InOrderCluster;

  public function mount(int|string $record): void
    {
        /** @property Order record */
        $this->record = $this->resolveRecord($record);
        $this->mountOrderCluster();

    }

   protected function loadPageRelations(): void
    {
        // Relations needed only for this page
        $this->record->load(['contact', 'profile']);
    }

    protected function getPageFormSections(): array
    {
        return [
            $this->getApplicationSection(),
            $this->getApplicantSection(),
            $this->getRecommendationSection(),
        ];
    }
```
10. Build each section form. I created these in section traits that I could include in the custom page



```
trait HasApplicationSection
{
    protected function getApplicationEditKey(): string
    {
        return 'editable-application-section';
    }

    protected function toggleApplicationEditMode()
    {

        $component = $this->form->getComponents()[0];
        $set = new Set($component);

        $set($this->getApplicationEditKey(), false);

    }

    public function saveApplication(): void
    {
        // Handle Validation
        $this->validateApplicationSection();

        // Get Application Form Data
        $data = $this->getApplicationSectionData();

        // Handle Record Update
        $this->handleApplicationRecordUpdate($data);

        // Handle Notification
        $this->handleApplicationNotification();

        // Reset Edit Mode
        $this->toggleApplicationEditMode();

    }

    protected function validateApplicationSection()
    {
        $livewire = $this->form->getLivewire();
        // Get all form rules
        $rules = $livewire->getRules();
        // Only validate against the rules needed for the section
        $onlyRules = ['data.meta.rent', 'data.meta.unit', 'data.property_id'];

        $livewire->validate(\Arr::only($rules, $onlyRules));
    }

    protected function getApplicationSectionData(): array
    {
        // We just need a filament component, the first one on the form will do
        $component = $this->form->getComponents()[0];
        $get = new Get($component);

        $meta = \Arr::only($get('meta'), ['rent', 'unit']);
        // We need to remove the state transformation for currency
        $meta['rent'] = CurrencyHelper::floatval($meta['rent']);

        return [
            ...$meta,
            'property_id' => $get('property_id'),
        ];
    }

    protected function handleApplicationRecordUpdate(array $data)
    {
        $this->record->meta->update(\Arr::only($data, ['rent', 'unit']));
        $this->record->update(['property_id' => $data['property_id']]);
    }

    protected function handleApplicationNotification()
    {
        Notification::make()
            ->title('Application Updated Successfully')
            ->icon('mdi-clipboard-text')
            ->iconColor('primary')
            ->success()
            ->send();
    }

    protected function sectionTitle(
        int $propertyId,
        string $unit,
        string $rent
    ): HtmlString {
        $propertyName = Property::find($propertyId)->name;

        $html = 'Application - '
            .'<span class="font-light text-sm">'.$propertyName.'</span>'
            .'<span class="text-gray-200 text-xs"> | </span>'
            .'<span class="font-light text-sm">'.$unit.'</span>'
            .'<span class="text-gray-200 text-xs"> | </span>'
            .'<span class="font-light text-sm">'.$rent.'</span>';

        return new HtmlString($html);
    }

    protected function getApplicationSection(): Component
    {
        $enableEditKey = $this->getApplicationEditKey();

        return Section::make(
            fn (Get $get) => $this->sectionTitle($get('property_id'), $get('meta.unit'), $get('meta.rent')
            ))
            ->headerActions([
                FormAction::make('edit-application')
                    ->label('Edit Application')
                    ->size(ActionSize::Small)
                    ->icon('mdi-clipboard-edit')
                    ->iconButton()
                    ->action(function (Set $set) use ($enableEditKey) {
                        $set($enableEditKey, true);
                    }),
            ])
            ->footerActions([
                FormAction::make('save')
                    ->label('Save')
                    ->size(ActionSize::Small)
                    ->icon('mdi-content-save')
                    ->hidden(function (Get $get) use ($enableEditKey) {
                        return ! $get($enableEditKey);
                    })
                    ->action(fn () => $this->saveApplication()),

                FormAction::make('cancel')
                    ->label('Cancel')
                    ->size(ActionSize::Small)
                    ->icon('mdi-close-circle')
                    ->color('gray')
                    ->hidden(function (Get $get) use ($enableEditKey) {
                        return ! $get($enableEditKey);
                    })
                    ->action(function (Set $set) use ($enableEditKey) {
                        $set($enableEditKey, false);
                    }),

            ])
            ->collapsible()
            ->icon('mdi-clipboard-text')
            ->schema([
                Hidden::make($enableEditKey)
                    ->dehydrated(false)
                    ->live(),
                Select::make('property_id')
                    ->label('Property')
                    ->selectablePlaceholder(false)
                    ->disabled(function (Get $get) use ($enableEditKey) {
                        return ! $get($enableEditKey);
                    })
                    ->suffixIcon('mdi-home-city')
                // ->native(false)
                    ->options(fn ($record) => Property::options($record->property->company_id)),
                TextInput::make('meta.unit')
                    ->extraInputAttributes(['x-on:keyup.enter' => '$wire.saveApplication()'])
                    ->suffixIcon('mdi-door-open')
                    ->label(function (Order $record) {
                        return $record->meta->hasOverride('unit')
                            ? new BadgeLabel('Unit')
                            : 'Unit';
                    })
                    ->helperText(function (Order $record) {
                        return $record->meta->hasOverride('unit')
                            ? 'Original Value: '.$record->meta->getOverride('unit')->old_value['value']
                            : '';
                    })
                    ->disabled(function (Get $get) use ($enableEditKey) {
                        return ! $get($enableEditKey);
                    }),
                TextInput::make('meta.rent')
                    ->suffixIcon('mdi-cash-marker')
                    ->extraInputAttributes(['x-on:keyup.enter' => '$wire.saveApplication()'])
                    ->label(function (Order $record) {
                        return $record->meta->hasOverride('rent')
                            ? new BadgeLabel('Rent')
                            : 'Rent';
                    })
                    ->helperText(function (Order $record) {
                        return $record->meta->hasOverride('rent')
                            ? 'Original Value: '.Number::currency($record->meta->getOverride('rent')->old_value['value'])
                            : '';
                    })
                    ->disabled(function (Get $get) use ($enableEditKey) {
                        return ! $get($enableEditKey);
                    })
                    ->formatStateUsing(fn ($state) => Number::currency($state)),

            ])->columns(3);
    }
}

```
