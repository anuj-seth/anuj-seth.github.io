---
layout: post
title: "Selenium and Bootstrap Modal Dialogs"
date: 2018-04-08
style: |
  body {
  font-size: 15px;
  }
---
Bootstrap modal dialogs that scroll across the screen from any direction were causing headaches for my automation code.  
I was seeing intermittent errors  "element not clickable at point (x,y). Other element would receive the click" while trying to click a button on a modal dialog.  
In my case there was a time lag between locating the element and clicking on it, during which the element location changed.  
I wrote a custom expected condition that waits for the element to stop moving. This expected condition can wait for the whole modal dialog to stop moving or any element inside it.  
```
class element_located_to_be_stationary(object):
    """An expectation that the element to be located is stationary.
    It is used to make sure bootstrap modal elements have stopped moving"""
    def __init__(self, locator):
        self.locator = locator
        self.last_position = None

    def __call__(self, driver):
        elt = driver.find_element(*self.locator)
        current_position = elt.location_once_scrolled_into_view
        print("last_position {0}, current_position {1}".format(self.last_position, current_position))
        if self.last_position is None or (current_position != self.last_position):
            self.last_position = current_position
            return False
        else:
            return elt

```
This expected condition can be used in your code as follows
```
# assuming that the element of interest is a button inside the modal dialog 
btn_in_modal_locator = (By.CSS_SELECTOR,
                        'button.btn')
wait = WebDriverWait(driver, 10)
btn_in_modal = wait.until(element_located_to_be_stationary(btn_in_modal_locator))
btn_in_modal.click()

```
