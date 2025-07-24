# LucaERP_11
The latest intallment in the LucaP ERP series. We build the hybrid template. 

# LUCAP_ERP11 The Hybrid template with predicates

#LucaP 

We're building an ERP from scratch and are currently working on three templates to do all the work. We've completed the form and the reporting view. Invoices, bills of materials, and other modules require a template that has three components: a form at the top called a *header*, a report in the middle known as the *body*, which contains details, and a form at the bottom called a *footer*. This time, we'll combine the two templates we have to create the third. Before we do, there's one improvement we'll make to the report: Adding predicates. 

# Predicates
A predicate is a Boolean expression that we use to filter our data. If true, it is valid data. Why you need them is easily seen in SQL. There are two places you'll commonly see a predicate: The `WHERE` clause and the `ON` clause. 

```
SELECT 
	t0.id,
	t0.name,
	t0.address, 
	t1.item,
	t1.quantity
FROM 
	Order T0,
	INNER JOIN OrderRow T1 ON t0.id = t1.parentId
WHERE
	t0.id = 123456
```

We've already looked at the predicate of `t0.id = '123456'` that's the R of CRUD, filtering for one record. In Swift, we wrote that as

```
row = table.first{$0.id == 123456}
```

Notice the braces that place the boolean expression in a closure. The `$0` is shorthand. Expanded out, we write this: 

```
row = table.first{row in row.id == 123456}
```

Let's look at the `ON` and what it does. It matches all cases where a child table has an `id` that is the same as the `parentID`. Graphically this is

`insert image here`


In Swift, instead of using `first`, as we did for our `WHERE` clause, we would use the filter method, which again takes a boolean closure for a predicate. 

```
rows = childTable.filter{$0.id == parentID}
```

This expression returns all rows relevant to the parent. While there are cases where reports require every row, most will be filtered to include only relevant information. A daily sales report will show only today's or yesterday's sales. They might be filtered to only sales bigger than $10,000. Alternatively, the report might be part of the hybrid report and give only child rows for the parent. 

For this to be flexible enough to work, we'll utilize a special feature of closures and their counterparts in other languages, lambdas and blocks: they can be assigned to variables and passed as arguments.

I can do this in Swift: 

```
var table:[Int] = [3,1,4,1,5,9,2,6,5,3,5,8,9,7]
var even:(Int)->Bool = {item in item % 2 == 0} \\<--- Predicate
var evens = table.filter(even)
print(evens)
``` 

This code returns only the even numbers. I declared my closure as

```
(type)-> Bool 
```

the same way I declare a `Int` or `String`. If this looks familiar, it is: a `func` is a form of a closure. 

```
func even(item:Int)->Bool{return item % 2 == 0}
```

To take this one more mind-blowing step, the function can be a function of predicates: 

```
 static func evenOdd(isEven:Bool = true) -> (Test)->Bool {
        if isEven{
            return {$0.a % 2 == 0}
        }
        return {$0.a % 2 != 0}
    }
```


In our hybrid template, we'll be filtering for the child view. If you recall several installments ago, we discussed child data, which typically includes the details of a parent data point. For example, a pizza recipe will have parent data about the pizza, and child records with the ingredients used. 

When we change rows on the parent, we show the corresponding rows of the child. Passing a predicate makes it much easier to set the correct data. 

# Updating the report template for hybrid 
In the last newsletter, I ran into a problem with `ForEach` loops and spent a column explaining them. I decided to start from scratch, writing the report instead of fixing all the errors that came with the existing report structure.  

I'll make a new `ReportUIView`. Most, but not all, of what I'm doing was discussed in earlier newsletters as we built the report and form templates. 

However, there are some significant changes, so I will focus on those.

## The New Model: David's Grindz
Up to now, for the templates, we had a flat model. This time, we need a parent-child relationship, a one-to-many setup. Taking a cue from the capstone project *David's Grindz* in my LinkedIn Learning course [The Complete Guide to SwiftUI](https://www.linkedin.com/learning/complete-guide-to-swiftui/faster-swiftui), I used a variation of a pizza recipe file. 

First, there's the parent, which adopted `Crudable`. Remember, `Crudable` is the protocol I used to handle most of my file functions.  

```
struct TestModelRow: Identifiable, Activatable, Equatable{
    var id: Int
    var isActive: Bool = true
    var name: String
}


class PizzaModel: Crudable{
    typealias Row = TestModelRow
    var table:[Row] = [
        TestModelRow(id: 0,isActive:true, name: "Huli Pizza"),
        TestModelRow(id: 1,isActive:true, name: "Longboard"),
        TestModelRow(id: 2,isActive:true, name: "Pepperoni Pizza"),
        TestModelRow(id: 3,isActive:true, name: "Margherita")
    ]
    var blank: Row{
        TestModelRow(id: -1, isActive: false, name: "")
    }
    
    var nextID: Int{
        (table.map{$0.id}.max{$0<$1} ?? -1 ) + 1
    }
    
}
```


For the child, there's one difference in the rows. I added a `parentID` to link back to the parent.  

```
struct IngredientModelRow: Identifiable, Activatable, Equatable{
    var id: Int
    var parentID:Int
    var isActive: Bool = true
    var name: String
}
```

I'm keeping this simple for now. I could create another column for the ingredient row and then build unique IDs from that. For now, I won't do that. 

I created an `IngredientModel` class that inherits from `Crudable` and includes some sample data. 

```
class IngredientModel: Crudable{
    typealias Row = IngredientModelRow
    var table:[Row] = [
        // With huli huli chicken, onions, ginger, crushed macadamia nuts, tomato sauce, and cheese on a classic crust.
        IngredientModelRow(id: 0, parentID: 0, name: "Hawaiian crust"),
        IngredientModelRow(id: 1, parentID: 0, name: "huli huli Chicken"),
        IngredientModelRow(id: 2, parentID: 0, name: "onions"),
        IngredientModelRow(id: 3, parentID: 0, name: "fresh ginger"),
        IngredientModelRow(id: 4, parentID: 0, name: "mac nuts"),
        IngredientModelRow(id: 5, parentID: 0, name: "tomato sauce"),
        IngredientModelRow(id: 6, parentID: 0, name: "firm mozzarella"),
        IngredientModelRow(id: 7, parentID: 0, name: "Huli Sauce"),
        //A very long flatbread for vegetarians and vegans, made with olive oil, mushrooms, garlic, fresh ginger, and macadamias, sweetened with lilikoi.
        IngredientModelRow(id: 8, parentID: 1, name: "classic rust"),
        IngredientModelRow(id: 9, parentID: 1, name: "olive oil"),
        IngredientModelRow(id: 10, parentID: 1, name: "mushrooms"),
        IngredientModelRow(id: 11, parentID: 1, name: "Garlic"),
        IngredientModelRow(id: 12, parentID: 1, name: "fresh ginger"),
        IngredientModelRow(id: 13, parentID: 1, name: "mac nuts"),
        IngredientModelRow(id: 14, parentID: 1, name: "lilikoi sauce"),
        //The New York Classic version. A thin crust with pizza sauce, cheese, and pepperoni
        IngredientModelRow(id: 15, parentID: 2, name: "New York crust"),
        IngredientModelRow(id: 16, parentID: 2, name: "firm mozzarella"),
        IngredientModelRow(id: 17, parentID: 2, name: "pepperoni"),
        IngredientModelRow(id: 18, parentID: 2, name: "pizza sauce"),
        //The classic pizza of Buffalo Mozzarella, tomatoes, and basil on a classic crust.
        IngredientModelRow(id: 19, parentID: 3, name: "classic crust"),
        IngredientModelRow(id: 20, parentID: 3, name: "tomatoes"),
        IngredientModelRow(id: 21, parentID: 3, name: "fresh basil leaves"),
        IngredientModelRow(id: 22, parentID: 3, name: "Buffalo mozzarella"),
    ]
    
    var blank: Row{
        Row(id: -1,parentID: -1, isActive: false, name: "")
    }
    
    var nextID: Int{
        (table.map{$0.id}.max{$0<$1} ?? -1 ) + 1
    }
}

```


## Calling Actions
Reports do not change rows the same way we select forms with navigation buttons. Instead, we click on the row to select it. These toolbars and the okay button in the bottom toolbar used a set of variables, `displayMode`, `navigationAction`, and `doAction`, to work.

For our new version of the report, I can directly use the buttons I need, which are very limited in comparison. This makes for much cleaner code. For example, to add a row, I do this:
 
```
Button{
    let newId = childModel.nextID
    let newRow = Row(id: newId, parentID: parentID, isActive: true, name: "")
    performAction(actionMode: .add, newRow: newRow)
} label:{
    Image(systemName: "plus")
}
```

I'm still using the `performAction` method from before, but a pared-down version. Instead of automatically setting a new row, I set it in the button's action. I save myself a lot of monitoring for changes. 

I decide which action to take with the `selectedChildRow` state variable. My displayed rows are in a label. A tap on a row selects the row, showing it as yellow.  

```
Button{
    selectedChildRow = row
}
label :{
//Display the row here-----
Hstack{
...
}
	.foregroundStyle(.black.opacity(row.isActive ? 1.0 : 0.5))
	.background(.yellow.opacity(selectedChildRow.id == row.id ? 1.0 : 0.01))
} 
```

There's also a dimming feature for inactive rows when displayed. Rows appear black when active, instead of the usual button tint color. 

## The Loop 
Encapsulating this button is the `ForEach`, iterating over the rows:

```
ForEach($childModel.table.filter({$row in row.parentID == parentID && (row.isActive || showInactive) })){$row in
...
}
```

There is a lot to unpack here. because I'm using binding varables in the `Textfield`s, this has to be binding, thus the `$childModel`, and the `$row in` 

In between all that is the predicate, and it is a big one, accomplishing two tasks.

```
{
$row in row.parentID == parentID 
&& 
(row.isActive || showInactive) 
}
```
 
The first half finds all rows that match the parent ID sent to us from the superview. I defined  parentID as 

```
@Binding var parentID:Int
```

The resulting collection consists only of ingredients from that recipe. 

The second part has a different use. I want to hide or dim any inactive rows. Inactivating rows is something we need to consider in accounting. We cannot delete a record in many circumstances due to auditing reasons, but we usually would like it to disappear from view. There are cases where seeing inactive records is useful, so the row has a flag `isActive`, and there is a second flag `showInactive`. If either of these is true, then the row displays. As we've seen, inactive rows will be at 50% opacity compared to active rows. 

## Layout problems
I also realized that we have some problems with the layout. In the last report template, I changed the modifiers for the input fields to include a `fieldWidth`and then used a formula to control that, with 25% allocated to the label and 75% to the field. However, I've realized making this completely manual is a better strategy. 

The `totalFieldWidth` variable becomes two variables:

```
//var totalFieldWidth: Double
var labelWidth: Double
var fieldWidth: Double
```

However, I didn't want to refactor all the code that currently uses `totalFieldWidth ` right now. I took an approach of two initializers, one for the legacy one and one for the new version. 

```
 //original with a twist. Used to be the default, but made for adding the modification below.
    init(label:String,value:String,isActive:Bool = true, totalFieldWidth:Double){
        self.label = label
        self.value = value
        self.isActive = isActive
        self.labelWidth = totalFieldWidth * (label.isEmpty ? 0 : 0.25)
        self.fieldWidth = totalFieldWidth * (label.isEmpty ? 1 : 0.75)
    }
    
    
//modification from original. This keeps columns for both labels and fields to be specified.
    init(label:String,value:String,isActive:Bool = true,labelWidth:Double,fieldWidth:Double){
        self.label = label
        self.value = value
        self.isActive = isActive
        self.labelWidth = labelWidth
        self.fieldWidth = fieldWidth
    }
```

Legacy code still works, and the more specific code will set up correctly. 

The new fields appear as follows in the `ReportUIView`.

```
//Show independent of selection, usually read-only columns
LPIntField(label: "", value: $row.parentID, isActive: false, labelWidth: 0, fieldWidth: 50)
LPIntField(label: "", value: $row.id, isActive: false, labelWidth: 0, fieldWidth: 50)
```

## Active and Inactive 
Note that these two rows have `isActive` set to `false`. We don't want these two columns changed by the user. Only the name field is allowed to change. 

Here you may see one of two strategies. Some like the strategy of having all fields open to change. This works well and makes changes easier for the user. It does take more memory to do so. I prefer only changing the selected row. The user selects the row, and then the chosen row allows interaction. 

For that, I use an `if` to check if the row is selected: 

```
 // Show when row is selected ---------------------------
if isSelectedRow(row){
	LPTextField(label: "", contents: $ingredient, isActive: row.isActive, labelWidth: 0, fieldWidth: 200)             
} else {
// display when not selected
	Text(row.name).frame(width:200)
}
```

## Deletion
To delete, we use a menu attached to the end of the row. Again, I'm leaving both the option of deleting and inactivating. I'll choose one over the other depending on circumstances. 

```
Menu {
//Deletion -----------------------------
	Button{
    performAction(actionMode: .delete, newRow: selectedChildRow)
	} label:{
    Label("Delete", systemImage: "trash")
}
//Inactivation -------------------------
	Button{
		toggleActive()
	} label:{
    Label(isActive ? "Inactivate" : "Activate", systemImage: "sleep")
	}
} label: {
    Image(systemName:"ellipsis.circle")
}
``` 

## Updating 
For updates, we take care of that when we select a different record. AS long as this is a valid record, update the record in the table, then fill the views for input with the new selected data. 

```
//Update the row when the selection changes-----------------------------------------------
        .onChange(of: selectedChildRow.id) { oldValue, newValue in
            if  oldValue >= 0{
                let newRow = Row(id: oldValue, parentID: parentID, isActive: isActive, name: ingredient)
                performAction(actionMode: .update, newRow: newRow)
                ingredient = selectedChildRow.name
            }
            ingredient = selectedChildRow.name
            isActive = selectedChildRow.isActive
        }
    }
```


# Scroll Views
The last major problem is scroll views. There are two different ways of setting them up in this code. The first option is to scroll through the report only, and the second is to scroll through the full form. The choice you make often depends on the device. If you are working on a desktop with ample screen real estate, scrolling through rows makes sense. On the other hand, tablets, especially ones that will take up half of the screen for an on-screen keyboard, will want to use a scrolling view for the entire form. 

I prefer the scrolling form. I'll add a vertical scroll view to the entire form. Some people might nest these, but I like the one scroll. 

I will, however, add a horizontal scroll view to the report view. This way, if there are more columns than screen real estate, you can scroll over to see more columns. 

We've come a far way and now have our templates. There's more to do here and a few bugs, but we'll address those next time, when we add persistence to the templates. 

# The Whole Code

I explained this rather haphazardly. If you understand Swift, you'll probably want to look at the full code. You can find everything in this [GitHub repository](https://github.com/MakeAppPiePublishing/LucaERP_11). 



