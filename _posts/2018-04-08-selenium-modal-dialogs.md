---
layout: post
title: "Selenium and Bootstrap Modal Dialogs"
date: 2018-04-08
---
Bootstrap modal dialogs scroll across the screen from any direction and can cause headaches for Selenium automation code.
If you have ever seen an "element not clickable at point (x,y). Other element would receive the click" seen while trying to click a button on a modal dialog then you have been bitten by this bug.
As the dialogs move across the screen and finally come to rest at a certain position one solution that presents itself is to wait for the modal dialog or some element inside it to stop moving. This can be achieved by using a custom expected condition.
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
driver = webdriver.Chrome()
driver.maximize_window()
driver.get("file:///C:/temp/modal.html")


open_modal = driver.find_element_by_css_selector('button[data-toggle=modal]')
open_modal.click()

modal_element = (By.CSS_SELECTOR, #'div.modal-content') 
                 'button.btn[data-dismiss=modal]')
wait = WebDriverWait(driver, 10)
#wait.until(EC.visibility_of_element_located(modal_element))
close_modal = wait.until(element_located_to_be_stationary(modal_element))
close_modal.click()


```
