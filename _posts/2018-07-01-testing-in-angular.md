---
layout: post
title: "Testing in Angular"
author: "Zakaria"
comments: true
---

# Introduction: 

One of the aspects that make Angular attractive is its comprehensiveness. Angular comes with all the features and components needed to develop, test, and go to production. In this post, I would like to go briefly through testing in Angular and how to implement unit and integration tests. Angular relies on `jasmine`, `karma`, and `protractor` frameworks for doing the testing. The following application will be used throughout the examples: [https://github.com/zak905/testing-demo-angular](https://github.com/zak905/testing-demo-angular)   

# Unit testing

As the name implies, unit testing is about testing single blocks separately. In Angular, this would be equivalent to testing single components. It can include testing the proper creation of the component and its inner DOM, proper styling, and also method outcomes and invocations. Using Angular cli, the unit tests can be run by executing `ng test`. Let's take as an example the `ExpenseFormComponent` component:

Template:

```
<div class="expense-container">
  <label for="amount" >Amount: </label>
  <input type="number" id="amount" [(ngModel)]="amount" />
  <label for="amountVAT">VAT: </label>
  <input type="number" id="amountVAT" [value]="amount * 0.2" [(ngModel)]="amountVAT" disabled/>
  <label for="currency">Currency: </label>
  <select id="currency" [(ngModel)]="selectedCurrency">
    <option *ngFor="let currency of currencies"> {{ currency }} </option>
  </select>
  <label for="date"  >Date: </label>
  <input type="date" id="date" [(ngModel)]="date"/>
  <label for="reason">Reason: </label>
  <textarea id="reason" [(ngModel)]="reason"> </textarea>
  <button id="addButton" (click)="addExpense($event)">Add Expense</button>
</div>
```

Implementation:

{% highlight ts %}

@Component({
  selector: 'app-expense-form',
  templateUrl: './expense-form.component.html',
  styleUrls: ['./expense-form.component.css']
})
export class ExpenseFormComponent implements OnInit {
  amount;
  //bug: this is always undefined
  amountVAT;
  date;
  reason;
  currencies = ["EUR", "USD"];
  selectedCurrency;

  @Output()
  expenseAdded: EventEmitter<object> = new EventEmitter<object>();

  constructor() { }

  ngOnInit() {
  }

  addExpense(event: Event) {
    console.log("clicked")
    this.expenseAdded.emit({amount: this.amount, amountVAT: this.amount * 0.2, date: this.date, selectedCurrency: this.selectedCurrency, reason: this.reason})
  }
}
{% endhighlight %}

a reasonable unit test would be:

{% highlight ts %}

it('should create the component', () => {
    expect(component).toBeTruthy();
  });

  it('should have 3 inputs, 5 labels, one select, one text area, and one button', () => {
  //  expect(component).toBeTruthy();
    const nativeElement = fixture.nativeElement;
    const inputs = nativeElement.querySelectorAll("input")
    expect(inputs.length).toEqual(3)
    const selects = nativeElement.querySelectorAll("select")
    expect(selects.length).toEqual(1)
    const textareas = nativeElement.querySelectorAll("textarea")
    expect(textareas.length).toEqual(1)
    const buttons = nativeElement.querySelectorAll("button")
    expect(buttons.length).toEqual(1)
    const labels = nativeElement.querySelectorAll("label")
    expect(labels.length).toEqual(5)
  });

  it('should have the correct styles', () => {
    const nativeElement = fixture.nativeElement;
    const elements = nativeElement.getElementsByClassName("expense-container")
    expect(elements.length).toEqual(1)
  });

  it('call emit on button click', () => {
    spyOn(component.expenseAdded, "emit");
    const nativeElement = fixture.nativeElement;
    const button = nativeElement.querySelector("button");
    button.dispatchEvent(new Event("click"));
    expect(component.expenseAdded.emit).toHaveBeenCalled();
  });

{% endhighlight %}

We have tested the creation of our component, its content, the styles, and finally we have tested if the `emit` method of the  `expenseAdded` event emitter is called. 

# Integration testing (or end-to-end testing)

Integration testing deals with the overall behavior of the application which implies all the components interacting together. During end-to-end testing, the test framework mimics the behavior of the end user and checks if the behavior of the app is the one expected. For example, the main behavior that we would like to test in our expense example app is that a button click should lead to the creation of a new expense and its appending to the table :

{% highlight ts %}

it('should add a new table row with expense details after submitting', () => {
    page.navigateTo();
    const amountInput = page.getAmountInput()
    amountInput.click()
    amountInput.sendKeys("70")
    const selectedCurrencySelect = page.getSelectedCurrencySelect()
    selectedCurrencySelect.click()
    browser.actions().sendKeys(protractor.Key.DOWN).perform();
    browser.actions().sendKeys(protractor.Key.ENTER).perform();
    const dateInput = page.getDateInput()
    dateInput.sendKeys("07")
    dateInput.sendKeys("03")
    dateInput.sendKeys("2018")

    const reasonInput = page.getReasonInput()
    reasonInput.sendKeys("travel")

    const addExpenseButton = page.getAddExpenseButton()
    addExpenseButton.click()

    page.getNewExpenseAmountCell().getText().then(amount => expect(amount).toEqual("70"))
    page.getNewExpenseAmountVATCell().getText().then(amountVAT => expect(amountVAT).toEqual("14"))
    page.getNewExpenseDateCell().getText().then(date => expect(date).toEqual("2018-07-03"))
    page.getNewExpenseCurrencyCell().getText().then(currency => expect(currency).toEqual("EUR"))
    page.getNewExpenseReasonCell().getText().then(reason => expect(reason).toEqual("travel"))
  });

  {% endhighlight %}

  end-to-end tests can run by executing `ng e2e`. 

# Wrap up

Testing is inevitable, and Angular team has understood that very well. Unlike other frameworks, Angular comes wrapped with the necessary tools required to test every aspect of an app. This could be a differentiating feature for Angular compared with other frameworks. We have seen in this post simple examples of unit testing and end-to-end testing in Angular, and those can be starting points towards more complex and comprehensive testing.    