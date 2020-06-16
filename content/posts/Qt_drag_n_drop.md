---
title: "Implementing drag and drop in Python using Qt"
date: 2020-06-14T20:55:41-05:00
draft: true
tags: ['python', 'Qt', 'PySide']
---

Lately, I needed to implement a drag and drop behaviour to reorder widgets in a tool and this article is just a reminder to my future self on how to do it and what caveats I should remember avoiding.


I had to implement this using a model/view approach with `Qt`.

Here is the official doc : [Qt Model/View ](https://doc.qt.io/qt-5/model-view-programming.html)

## Let's get started !

So first let's look at what the `model` needs in order to implement this correctly.

It needs to have the following methods (check the links for more information):

- [`insertRows()`](https://doc.qt.io/qt-5/qabstractitemmodel.html#insert) : important not to use the `insertRow()` method. Qt's documentation sepcifies that `insertRows()` method should be implemented rather than this one.
- [`removeRows()`](https://doc.qt.io/qt-5/qabstractitemmodel.html#removeRows)
- [`mimeData()`](https://doc.qt.io/qt-5/qabstractitemmodel.html#mimeData)
- [`mimeTypes()`](https://doc.qt.io/qt-5/qabstractitemmodel.html#mimeTypes)
- [`dropMimeData()`](https://doc.qt.io/qt-5/qabstractitemmodel.html#dropMimeData)
- [`flags()`](https://doc.qt.io/qt-5/qabstractitemmodel.html#flags)
- [`supportedDropActions()`](https://doc.qt.io/qt-5/qabstractitemmodel.html#supportedDropActions)

### Starting point : mimeTypes()

First, we need to reimplement the `mimeTypes()` method to let our model know that
it will only need to act if a certain type of data is dragged or dropped.
By default, the built-in models and views use an internal MIME type: application/x-qabstractitemmodeldatalist.

```python
class FoobarModel(QtCore.QAbstractListModel):
    MIME_TYPES_ACCEPTED = 'application/foobar_data'
    # ...
    def mimeTypes(self):
        """Return a list of MIME types used to describe a list of indices.
        Returns:
            list(str): A list of accepted mime types.
        """
        return [self.MIME_TYPES_ACCEPTED]
```
Since we defined a custom `mimeTypes()` we need to implement the `mimeData()` and `dropMimeData()` functions as well.

According to the documentation :
```
Returns an object that contains serialized items of data
corresponding to the list of indexes specified.
The format used to describe the encoded data is obtained from the mimeTypes()
function.
```

So we need to create an object that will contain serialized data and that is the `QMimeData` object.

According to the documentation again :

```
QMimeData is used to describe information that can be stored in the clipboard,
and transferred via the drag and drop mechanism.
QMimeData objects associate the data that they hold with the corresponding MIME
types to ensure that information can be safely transferred between applications
and copied around within the same application.
```

Its data is stored as `QByteArray`.

```python
def mimeData(self, indexes):
    mime_data = QtCore.QMimeData()
    data = QtCore.QByteArray()

    for index in indexes:
        if index.isValid():
            data.append("/{0}".format(index.row()))

    mime_data.setData(self.MIME_VERSION, data)

    return mime_data
```

The data is stored as a plain string.
It can be a good idea to use a delimiter. That way we can store data for multiple rows and split it when processing the drop.


### What kind of drop can I do?

We need to also define in the model what kind of drop it supports.

The default is the `QtCore.Qt.CopyAction` but I wanted to move things around hence the use of :

```python
def supportedDropActions(self):
    return QtCore.Qt.MoveAction
```

### What can be dragged ?

To define which items can be dragged we need to implement the `flags()` method.

```python
def flags(self, index):
    flags = super(FoobarModel, self).flags(index)
    if not index.isValid():
        return QtCore.Qt.ItemIsEnabled | QtCore.Qt.ItemIsDropEnabled | flags
    return QtCore.Qt.ItemFlags(QtCore.Qt.ItemIsDragEnabled | flags)
```

With this we are doing two things :

- We first grab the defaults flags from the model.
- We enhance the default flags to add the `QtCore.Qt.ItemIsDragEnabled`
- Then if the index is not valid (meaning we drop outside the items in the list)
then its flags are extended to support it with `QtCore.Qt.ItemIsDropEnabled`

### Insert rows and remove them

The next thing we need to implement now is the insertion and removal of the rows :

```python
def insertRows(self, row, count=0, parent=QtCore.QModelIndex()):
    self.beginInsertRows(parent, row, row + 1)
    self._data.append('Foo {0}'.format(str(row).zfill(4)))
    self.endInsertRows()
    return True

def removeRows(self, row, count=1, parent=QtCore.QModelIndex()):
    self.beginRemoveRows(parent, row, row + count - 1)
    self._data.pop(row)
    self.endRemoveRows()
    return True
```


### Drop the stuff

The dropping of items is handled by the `dropMimeData()` function.

Quick note, if you drop things at the very end of the list, the index value might
be `-1`, so in the case of our model it needs to be changed to the length of the
list.

The idea here is to :

- List the indexes of the items that are being moved.
- Determine the row where we want to drop.
- Create a list of items in reversed order to insert at the previously determined row.
- Remove the selected rows at their current index.
- Insert them at their new indexes.

```python
def dropMimeData(
        self,
        data,
        action,
        row,
        column=0,
        parent=QtCore.QModelIndex()):
    if action is QtCore.Qt.IgnoreAction:
        return False

    indexes_of_items_to_move = [
        i.toInt()[0] for i in data.data(self.MIME_VERSION).split('/')[1:]
    ]
    begin_row = row
    dropping_outside = False
    for item_row in indexes_of_items_to_move:
        if item_row < row:
            begin_row -= 1
    if row == -1:
        begin_row = self.rowCount() - 1
        dropping_outside = True

    items_to_move = sorted(
        [self._data[i] for i in indexes_of_items_to_move],
        reverse=True,
    )

    for item in items_to_move:
        self.beginRemoveRows(
            parent, self._data.index(item), self._data.index(item),
        )
        self._data.pop(self._data.index(item))
        self.endRemoveRows()

    if dropping_outside:
        begin_row = self.rowCount()
    self.beginInsertRows(
        QtCore.QModelIndex(), begin_row, begin_row + len(items_to_move) - 1
    )
    for item in items_to_move:
        self._data.insert(begin_row, item)
    self.endInsertRows(),

    first = self.index(begin_row, 0)
    last = self.index(begin_row + len(items_to_move), 0)
    self.dataChanged.emit(first, last)

    return False
```



## Catch and Caveats

### I drop  my items, but some of my rows are being removed !

If you take care of the reordering in the `dropMimeData()` method, be sure to not return `True` at the end as the Qt documentation is adivsing.
If you return `True` then the `removeRows()` method gets called as the model data has changed.
And you will end up with rows getting removed although you just wanted to move them.


## BONUS Points:

If you want to have a custom indicator of the widgets you are dragging, you need to reimplement the method `startDrag()` on the view.

We'll see that next time !


{{<gist victorfleury d895872f0ae7cbff41db636c6e70ae55>}}
