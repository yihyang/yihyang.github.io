---
layout: post
title:  "Modeling Workflow"
date:   2020-02-24 00:40:00 +0800
categories: [blog]
tags: [application, database design, laravel, system design, workflow]
---

*(P.S. This is a WIP but I will try to complete it soon)*

Recently I noticed that there have been a few occurrence where we need to design a series of steps which allowed users to perform the same actions repeatedly.

The actions are mostly related to the business flow which are not limited to the followings:
- A flow that contained a few steps
- User enter some data in each step
- Another user in a group review / approve it
- With enough approvals the state will change automatically
- Sometimes when the step changed to a certain state it will automatically create a new step
- Other times different steps will be created based on the input inside the data

The repeated pattern looks like a workflow to me and it made me think - *What if we can abstract the repeated patterns and put it into a database instead of code?*
By putting the steps into database it will make the flow configurable and therefore reduces the effort (e.g. code changes) needed whenever we need to have a new workflow.

## Credits

Before I start I would like to thank [Mathew Jones][mathew-jones] for his article on [how to create a workflow engine database][workflow-engine-database] for inspirations.

Also I would like to thank [Jeremy Wadhams][jeremy-wadhams] for his awesome plugin [JsonLogic][json-logic-website] that empowered the development of this module.

## Prerequisite

I'm assuming you have used [Laravel][laravel-website] before or you should at least have some understand of Object Oriented Programming and Model-View-Controller framework.

## Repo

I have build an application based on the implementation mentioned here. You can checkout the repo [here][laravel-workflow-demo].

## Assumptions

I'm making the following assumption about the workflow:
- Starting the workflow
    - A model can have multiple workflows
    - There is a set of condition that allows the workflow to be started
    - The workflow can be started automatically or manually
- Creating different process within the workflow
    - A workflow can contain multiple steps
    - The steps' state can be changed based on different conditions.
    - TODO: Add on here
- At the end of the workflow, the attached module's status will be updated.
    - TODO: Add on here

I will try to use a case study for a fictional bank - **E Corp Bank** in order to help understanding the implementation.

I will also write test in order to validate that the features are functioning.

## Starting the workflow

The workflow should be able to start by it's own based on a trigger.

### Problem Statement

**E Corp Bank** wanted to automated their workflow for their Transaction which included a process of few steps that included approvals and change of states based on different condition.

1. When a new transaction with the type "Personal Loan" is selected, and:
    1. If the amount is less than $10,000, it will start the "Low Loan Amount" workflow
    2. If the amount is more than or equal to $10,000 but less than $50,000, it will start the "Medium Loan Amount" workflow
    3. If the amount is more than or equal to $50,000, it will start the "High Loan Amount" workflow

### Generate Models & Migrations

#### Transaction

First, we will generate a Transaction module in order for us to demonstrate the functionality of workflow

Run the following command to generate a new model named Transaction

```
php artisan make:model Transaction -m
```

The following attributes are added for Transaction model:

| Name | Type | Description |
|---|---|---|
| type | string | The type of the transaction |
| data | json | Data stored for transaction |


##### Model

{% capture code-transaction-model %}
```php
class Transaction extends Model
{
    use HasWorkflowTrait;

    protected $fillable = [
        'type',
        'data'
    ];

    protected $casts = [
        'data' => 'json'
    ];
}
```
{% endcapture %}

{% include widgets/toggle-field.html toggle-name="toggle-code-transaction-model" toggle-text=code-transaction-model %}

##### Migration

{% capture code-transactions-migration %}
```php
Schema::create('transactions', function (Blueprint $table) {
    $table->bigIncrements('id');

    $table->string('type')->nullable();
    $table->json('data')->nullable();

    $table->timestamps();
});
```
{% endcapture %}

{% include widgets/toggle-field.html toggle-name="toggle-code-transactions-migration" toggle-text=code-transactions-migration %}

#### Workflow Template

Next, we will generate **Workflow Template** model that will allow workflow to be created.

The **Workflow Template** model will contain the following attributes:

| Name | Type | Description |
|---|---|---|
| name | string | The name of the workflow |
| model_name | string | The name of the model that this workflow will be attached to |
| start_condition | json | condition which allow the workflow to be started |
| autostart | boolean | Whether the workflow will be started by it's own. |

##### Model

{% capture code-workflow-template-model %}
```php
use Illuminate\Database\Eloquent\Model;

class WorkflowTemplate extends Model
{
    protected $fillable = [
        'autostart',
        'model_name',
        'name',
        'start_condition',
    ];
    protected $casts = [
        'start_condition' => 'json'
    ];

    /**
     * workflows that this template has produced
     *
     * @return HasMany
     */
    public function workflows()
    {
        return $this->hasMany(Workflow::class);
    }
}
```
{% endcapture %}

{% include widgets/toggle-field.html toggle-name="toggle-code-workflow-template-model" toggle-text=code-workflow-template-model %}

##### Migration

{% capture code-workflow-templates-migration %}
```php
Schema::create('workflow_templates', function (Blueprint $table) {
    $table->bigIncrements('id');

    $table->string('name')->nullable();
    $table->string('model_name')->nullable();
    $table->json('start_condition')->nullable();
    $table->boolean('autostart')->nullable();

    $table->timestamps();
});
```
{% endcapture %}

{% include widgets/toggle-field.html toggle-name="toggle-code-workflow-templates-migration" toggle-text=code-workflow-templates-migration %}

#### Workflow

We also have a **Workflow** model to record the actual workflow that was created.

The **Workflow** model will contain the following attributes:

| Name | Type | Description |
| originator_type | string | The name of the model that this workflow started |
| originator_id | integer | The id of the entity that this workflow attached to |
| template_id | integer | The template that this workflow referred from |
| state | string | The state of the workflow |


##### Model

{% capture code-workflow-model %}
```php
use Illuminate\Database\Eloquent\Model;

class Workflow extends Model
{
    protected $fillable = [
        'originator_id',
        'originator_type',
        'state',
        'workflow_template_id'
    ];

    /**
     * The template that this workflow referred from
     */
    public function template()
    {
        return $this->belongsTo(WorkflowTemplate::class);
    }
}

```
{% endcapture %}

{% include widgets/toggle-field.html toggle-name="toggle-code-workflow-model" toggle-text=code-workflow-model %}


##### Migration

{% capture code-workflow-migration %}
```php
Schema::create('workflows', function (Blueprint $table) {
    $table->bigIncrements('id');

    $table->string('originator_type')->nullable();
    $table->integer('originator_id')->nullable();
    $table->integer('workflow_template_id')->nullable();
    $table->string('state')->nullable();

    $table->timestamps();
});
```
{% endcapture %}

{% include widgets/toggle-field.html toggle-name="toggle-code-workflow-migration" toggle-text=code-workflow-migration %}

### Seeding data

We will be seeding the database with the following data (based on the assumptions on top):

The **start_condition** is created based on the JsonLogic which you can refer [here][json-logic-website]

1. Low Loan Amount
```php
WorkflowTemplate::firstOrCreate([
    'name'            => 'Low Loan Amount',
    'model_name'      => Transaction::class,
    'autostart'       => true,
    'start_condition' => [
        'and' => [
            [
                '==' => [['var' => 'type'], 'Personal Loan'],
            ],
            [
                '<' => [['var' => 'data.amount'], 10000],
            ],
        ],
    ],
]);
```

2. Medium Loan Amount
```php
WorkflowTemplate::firstOrCreate([
    'name'            => 'Medium Loan Amount',
    'model_name'      => Transaction::class,
    'autostart'       => true,
    'start_condition' => [
        'and' => [
            [
                '==' => [['var' => 'type'], 'Personal Loan'],
            ],
            [
                'and' => [
                    [
                        '>=' => [['var' => 'data.amount'], 10000]
                    ],
                    [
                        '<'  => [['var' => 'data.amount'], 50000]
                    ],
                ],
            ],
        ],
    ],
]);
```

3. High Loan Amount
```php
WorkflowTemplate::firstOrCreate([
    'name'            => 'High Loan Amount',
    'model_name'      => Transaction::class,
    'autostart'       => true,
    'start_condition' => [
        'and' => [
            [
                '==' => [['var' => 'type'], 'Personal Loan'],
            ],
            [
                '>=' => [['var' => 'data.amount'], 50000],
            ],
        ],
    ],
]);
```

### Traits

In order to DRY up code, I will be using trait in order to allow the models to attach the relationships and also adding related events.

I will be naming this trait as `HasWorkflowsTrait`

The trait needs to contain 2 things:
1. The relationship between the model and workflow
    - This is pretty straightforward as the model (In our case, Transaction) has one or many workflows

    ```php
    public function workflows()
    {
        return $this->morphOne(Workflow::class, 'originator');
    }
    ```

2. The event binding that will allow the auto start of the workflow, for that I will be doing the booting booting eloquent model traits, there's a pretty good blog that explained the use of it which you can find it [here][booting-eloquent-model-traits-blog]
    1. I wanted to binding the event to the "created" event of the model and therefore it will look like the following
        ```php
        static::created(function ($entity) {
            // code here
        })
        ```
    2. On the creation of the Transaction it will need to perform 3 tasks:
        1. Find the workflow template that is related to this module which has the **autostart** set to true
            ```php
            // get the name of the model
            $modelName = get_class($entity);
            // find the auto start workflow templates
            $templates = WorkflowTemplate::where([
                'model_name' => $modelName,
                'autostart'  => true,
            ])->get();
            ```
        2. Filter the workflow that fulfilled the condition
            ```php
            $autostartWorkflowTemplates = $templates->filter(function ($template) use ($entity) {
                $logic = $template->start_condition;
                $data  = $entity->toArray();

                // apply the logic with the data
                $result = JsonLogic::apply($logic, $data);

                // logic is wrong therefore it didn't return boolean
                if (!is_bool($result)) {
                    return false;
                }

                return $result;
            });
            ```

        3. Finally, create the workflow
```php
// create the workflow
$autostartWorkflowTemplates->each(function ($template) use ($modelName, $entity) {
    Workflow::create([
        'state'                => 'started',
        'originator_type'      => $modelName,
        'originator_id'        => $entity->id,
        'workflow_template_id' => $template->id,
    ]);
});
```


### Try It Out

With the above code in place, we should be able to see the workflow automatically created on creation of a new Transaction if it fits the criteria.

Execute tinker with the following

```
php artisan tinker
```

and run the following command

```php
Transaction::create(['type' => 'Personal Loan', 'data' => ['amount' => 1000]])
```

and you should be able to see the related workflow being created.

### Test

In *Working Effectively with Legacy Code* by Michael Feathers, he introduced a definition of legacy code as **code without tests**. Personally I strive to write test whenever possible to prove that the code works properly.

Let's test a few possible scenario here:

1. The workflow with correct condition and autostart flag set to "true" will be created automatically
    {% capture code-testcase-1 %}
```php
public function test_workflow_with_correct_condition_and_autostart_flag_set_to_true_will_be_created_automatically()
{
    $this->workflowTemplate = WorkflowTemplate::create([
        'name'            => 'Automated Workflow 1',
        'model_name'      => Transaction::class,
        'autostart'       => true,
        'start_condition' => [
            'and' => [
                [
                    '==' => [['var' => 'type'], 'Personal Loan'],
                ],
                [
                    'and' => [
                        [
                            '>=' => [['var' => 'data.amount'], 10000],
                        ],
                        [
                            '<' => [['var' => 'data.amount'], 50000],
                        ],
                    ],
                ],
            ],
        ],
    ]);

    $transaction = Transaction::create([
        'type' => 'Personal Loan',
        'data' => [
            'amount' => 12345,
        ],
    ]);

    $this->assertDatabaseHas(
        'workflows',
        [
            'originator_type'      => Transaction::class,
            'originator_id'        => $transaction->id,
            'workflow_template_id' => $this->workflowTemplate->id,
            'state'                => 'started',
        ]
    );
}
```
    {% endcapture %}
    {% include widgets/toggle-field.html toggle-name="toggle-code-testcase-1" toggle-text=code-testcase-1 %}

2. The workflow with wrong condition and autostart flag set to "true" will not be created automatically
    {% capture code-testcase-2 %}
```php
public function test_workflow_with_wrong_condition_and_autostart_flag_set_to_true_will_not_be_created_automatically()
{
    $this->workflowTemplate = WorkflowTemplate::create([
        'name'            => 'Automated Workflow 2',
        'model_name'      => Transaction::class,
        'autostart'       => true,
        'start_condition' => [
            'and' => [
                [
                    '==' => [['var' => 'type'], 'Personal Loan'],
                ],
                [
                    'and' => [
                        [
                            '>=' => [['var' => 'data.amount'], 10000],
                        ],
                        [
                            '<' => [['var' => 'data.amount'], 50000],
                        ],
                    ],
                ],
            ],
        ],
    ]);

    $transaction = Transaction::create([
        'type' => 'Housing Loan',
        'data' => [
            'amount' => 1,
        ],
    ]);

    $this->assertDatabaseMissing(
        'workflows',
        [
            'originator_type'      => Transaction::class,
            'originator_id'        => $transaction->id,
            'workflow_template_id' => $this->workflowTemplate->id,
            'state'                => 'started',
        ]
    );
}
```
    {% endcapture %}
    {% include widgets/toggle-field.html toggle-name="toggle-code-testcase-2" toggle-text=code-testcase-2 %}

3. The workflow with correct condition and autostart flag set to false will not be created automatically
    {% capture code-testcase-3 %}
```php
public function test_workflow_with_correct_condition_and_autostart_flag_set_to_false_will_not_be_created_automatically()
{
    $this->workflowTemplate = WorkflowTemplate::create([
        'name'            => 'Automated Workflow 3',
        'model_name'      => Transaction::class,
        'autostart'       => false,
        'start_condition' => [
            'and' => [
                [
                    '==' => [['var' => 'type'], 'Personal Loan'],
                ],
                [
                    'and' => [
                        [
                            '>=' => [['var' => 'data.amount'], 10000],
                        ],
                        [
                            '<' => [['var' => 'data.amount'], 50000],
                        ],
                    ],
                ],
            ],
        ],
    ]);

    $transaction = Transaction::create([
        'type' => 'Housing Loan',
        'data' => [
            'amount' => 12345,
        ],
    ]);

    $this->assertDatabaseMissing(
        'workflows',
        [
            'originator_type'      => Transaction::class,
            'originator_id'        => $transaction->id,
            'workflow_template_id' => $this->workflowTemplate->id,
            'state'                => 'started',
        ]
    );
}
```
    {% endcapture %}
    {% include widgets/toggle-field.html toggle-name="toggle-code-testcase-3" toggle-text=code-testcase-3 %}

4. The workflow with wrong condition and flag set to false will not be created automatically
    {% capture code-testcase-4 %}
```php
public function test_workflow_with_wrong_condition_and_autostart_flag_set_to_false_will_not_be_created_automatically()
{
    $this->workflowTemplate = WorkflowTemplate::create([
        'name'            => 'Automated Workflow 4',
        'model_name'      => Transaction::class,
        'autostart'       => false,
        'start_condition' => [
            'and' => [
                [
                    '==' => [['var' => 'type'], 'Personal Loan'],
                ],
                [
                    'and' => [
                        [
                            '>=' => [['var' => 'data.amount'], 10000],
                        ],
                        [
                            '<' => [['var' => 'data.amount'], 50000],
                        ],
                    ],
                ],
            ],
        ],
    ]);

    $transaction = Transaction::create([
        'type' => 'Housing Loan',
        'data' => [
            'amount' => 1,
        ],
    ]);

    $this->assertDatabaseMissing(
        'workflows',
        [
            'originator_type'      => Transaction::class,
            'originator_id'        => $transaction->id,
            'workflow_template_id' => $this->workflowTemplate->id,
            'state'                => 'started',
        ]
    );
}
```
    {% endcapture %}
    {% include widgets/toggle-field.html toggle-name="toggle-code-testcase-4" toggle-text=code-testcase-4 %}

That's it for now. In the next post I will talk about steps and transitions between states.


[laravel-website]: https://laravel.com/
[booting-eloquent-model-traits-blog]: https://www.archybold.com/blog/post/booting-eloquent-model-traits
[jeremy-wadhams]: https://github.com/jwadhams
[json-logic-website]: http://jsonlogic.com/
[laravel-workflow-demo]: https://github.com/yihyang/laravel-workflow-demo
[mathew-jones]: https://exceptionnotfound.net/author/matthew-jones/
[workflow-engine-database]: https://exceptionnotfound.net/designing-a-workflow-engine-database-part-1-introduction-and-purpose/
