---
layout: post
title:  "Changing the size and class of a selected TableViewCell"
date:   2017-06-26 21:51:00
tags: swift UIKit UITableView UITableViewCell
summary: >
  Changing the size and class of a selected TableViewCell

---

I had to cobble this together out of a bunch of StackOverflow
[answers](https://stackoverflow.com/a/2063776/1115020) so...

I have a UITableView where I want to change the appearance of a cell when it's selected.
I want to change both the height of the cell, _and_ the UITableViewCell class used.

It turned out that I had to do all of these...


{% highlight swift %}

var selectedIndexPath: IndexPath?

override func tableView(_ tableView: UITableView, willSelectRowAt indexPath: IndexPath) -> IndexPath? {
    if selectedIndexPath != indexPath {
        // don't do this if this was already the selected row
        selectedIndexPath = indexPath
        tableView.reloadRows(at: [indexPath], with: .none)  // animation will be handled later
    }
    return indexPath
}

override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
    // force the tableView to reset the height of the selected cell and animate it
    tableView.beginUpdates()
    tableView.endUpdates()
}

override func tableView(_ tableView: UITableView, didDeselectRowAt indexPath: IndexPath) {
    if selectedIndexPath == indexPath {
        selectedIndexPath = nil
    }
    // reloading the row will change both height and class, and also clears the selection
    tableView.reloadRows(at: [indexPath], with: .none)
}

override func tableView(_ tableView: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat {
    if indexPath == selectedIndexPath {
        return SELECTED_HEIGHT
    }
    return NORMAL_HEIGHT
}

override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    if indexPath == tableView.indexPathForSelectedRow || indexPath == selectedIndexPath {
        return tableView.dequeueReusableCell(withIdentifier: "SelectedCell", for: indexPath) as? SelectedTableViewCell
    }
    return tableView.dequeueReusableCell(withIdentifier: "NormalCell", for: indexPath) as? NormalTableViewCell
}

{% endhighlight %}
