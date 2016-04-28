Sending basic JSON data made out of strings to an API is, in most cases, a simple and straightforward task. But what about sending files, which are made of numerous lines of binary data and consits of various formats? Such data requires a slightly different approach and there are several formats through which files can be sent to the API.

In this guide, the two main approaches of handling file upload, base64 encoding and multipart form data, will be examined. The approaches will be implemented in a Rails 5 API application using both the [paperclip](https://github.com/thoughtbot/paperclip) and the [carrierwave](https://github.com/carrierwaveuploader/carrierwave) gems. A sample AngularJS application will also be written in order to present how the uploads can be handled client-side.

## Approaches for sending files to a Rails 5 API 
### Multipart form data
  Multipart forms have been around since [HTML 4](https://www.w3.org/TR/html401/interact/forms.html#h-17.13.4). They were introduced because the standard *application/x-www-form-urlencoded* forms did not handle bigger amounts of data well enough. According to [W3C](https://www.w3.org/)'s definition of *multipart/form-data*, it is used for forms that "contain files, non-ASCII data, and binary data". What multipart forms do differently is that they break the data in different chunks that represent the various charactereistics of the file (size, content type, name, contents) as well as the characteristics of the standard text and number data that is also sent with the form.
  
  To make things clearer, let's consider the following form:
  
  
```html
   <form action="http://localhost:3000/api/v1/items"
         enctype="multipart/form-data"
         method="post">
   <p>
   What is your name? <input type="text" name="submit-name"><br>
   What file are you sending? <input type="files" name="file"><br>
   </p>
   <input type="submit" value="Send"> <input type="reset">
 </form>
 
```
If the user enters "John" in the text input, and selects the text file "file1.txt", the user agent might send back the following data:

```
   Content-Type: multipart/form-data; boundary=AaB03x

   --AaB03x
   Content-Disposition: form-data; name="submit-name"

   Larry
   --AaB03x
   Content-Disposition: form-data; name="files"; filename="file1.txt"
   Content-Type: text/plain

   ... contents of file1.txt ...
   --AaB03x--

```
Every part of the form is separated by a boundary, which reprsents a different string (<code>AaB03x</code> in the example). Each parts contains information about the <code>Content-Type</code> it contains and the content of the part itself. Larger files can be broken down in chuncks and assembled in the server, enabling a file to be streamed and to have its integrity maintaned in cases of connection interruprtion.

Let's consider uploading another file. If the user selects another file  "file2.gif", the browser will construct the parts as follows:
```
   Content-Type: multipart/form-data; boundary=AaB03x

   --AaB03x
   Content-Disposition: form-data; name="submit-name"

   Larry
   --AaB03x
   Content-Disposition: form-data; name="file"
   Content-Type: multipart/mixed; boundary=BbC04y

   --BbC04y
   Content-Disposition: file; filename="file1.txt"
   Content-Type: text/plain

   ... contents of file1.txt ...
   --BbC04y
   Content-Disposition: file; filename="file2.gif"
   Content-Type: image/gif
   Content-Transfer-Encoding: binary

   ...contents of file2.gif...
   --BbC04y--
```

Here, it can be seen that there is another part added to the <code>form-data</code>. This time, since the file is not in a <code>text/plain</code> format, it is broken down in binary, as it can be inferred from the <code> Content-Transfer-Encoding</code> property. The <code>Content-Type</code> property gives information about the type of the file, also known as its [media (MIME) type](https://en.wikipedia.org/wiki/Media_type). If the <code> Content-Type </code> property  is not defined and  the file is not in text format, its format will default to <code> application/octet-stream </code> which means that the finaly is binary and has no type. When sending data to an API, it is always good to include a <code>Content-Type</code> to each part which contains a file, otherwise there would be no way to validate the contents of the file.
### Base64 encoding
Base64 for is one of the most commonly used binary to text encoding formats. It uses an algorithm to break up binary code in pieces and convert it in ASCII characters (text). It has a wide array of applications - apart from being used to encode files into text in order to send them to an API, it is also used represent images as a content soruce in CSS, HTML and SVG. 
The structure of base64 encoded files is very simple. It consits of two parts - a MIME type (similar to the multipart <code>Content-Type</code> and the actual base64 encoded string:
```
data:image/gif;base64,iVBORw0KGgoAAAANag...//rest of the base64 text
```
Usually, a file is encoded into base64 on the client and decoded on the server. The base64 string can be easily attached to a JSON object's attribute:
```javascript
 {
  "file": {
    "name": "file2",
    "contents": "data:image/gif;base64, iVBORw0KGgoAAA..."
  }
 }
```

An API would easily be able to pick up the parameter and decode it back to binary. This makes base64-encoded files uploads convenient for APIs, since the format of  the message containing the file and the way it is transferred does not differ from the standard way messages are sent to an API.


### Base64 encoding vs. Multipart form data
Base64 encoded files are easy to be used by JSON and XML APIs since they are represented as text and can be easily sent through the standard <code> application/json</code> format. However, the encoding increases the file size by 33%, making if difficult to transfer larger files. Encoding and decoding also adds a computational overhead for both the server and the client. Therefore, base64 is suitable for sending images and small files under 100MB. Multipart forms, on the other hand, are more "unnatural" to the APIs, since the data is encoded in a different format and requires a different way of handling. However, the increased performance with larger files and the ability to stream files makes multipart form data more desirable for uploading larger files such as videos.

## Setting up Rails API for file upload

 Let's apply these concepts into a real application. First, let's set up a Rails 5 API application and scaffold a model that is going to be used. 
 
### Installing and configuring Rails 5 API
 
To create a Rails 5 API, you need Ruby 2.2.4 and up installed. After you have a suitable Ruby version, the first step is to install the newest version of Rails using the following command in your terminal/command prompt:

```bash
gem install rails --pre --no-ri --no-rdoc
```

This will give you the ability to run <code> rails new </code> using the most recent version:

```bash
rails _5.0.0.beta3_ new fileuploadapp --api
```

This will create a brand new Rails 5 app, and with the <code> --api </code> option included in the generator, it will be all set up to be used as an API. Move to the directory of the new project:

```bash
 cd fileuploadapp
```

Go to the Gemfile and uncomment <code> gem jbuilder </code>.
```ruby
# Gemfile.rb
 gem 'jbuilder', '~> 2.0' # <-- this must be in your gemfile
```   

[JBuilder](https://github.com/rails/jbuilder) is used to create JSON structures for the responses from the application. In MVC terms, the JSON respones are going to be the view layer of the application and all Jbuilder-generated responses have to be put in <code> app/views/(view for a particular controller action) </code>. 
Install the gem:
```bash
bundle install
```
  With the gem installed, the model is going to be scaffolded with Jbuilder-generated views.
```bash
rails g scaffold Item name:string description:string
```
A model named <code>Item</code> is going to be scaffolded which has a name and description as strings. Note that there is still no information about the file - it will be included in the model's schema at a later stage. For now, we are all set to go ahead and migrate the model into the database:

```bash
rails db:migrate
```
This will create a table in the database for the new model. Note that one of the new features in Rails 5 is that you can use <code> rails </code> instead of <code> rake </code> for executing a migration command.
 

 The two most popular gems for uploading files are [Paperclip](https://github.com/thoughtbot/paperclip) and [Carrierwave](). The functionality of both is similar, so you can pick either of them. If you would like to try both approaches, it is recommended that you use version control such as [git](https://git-scm.com/) so that the implementations don't clash.
##  File upload using Paperclip
### Uploading a single file 
 Include the gem in your gemfile:
 
 ```ruby
 #Gemfile.rb
 gem "paperclip", "~> 5.0.0.beta1"
```
  And install it:
 ```bash
 bundle install
```  

 Generate a migration that will add the attachment to the databse. ** You have to put the same name for the attachment in the model as the one you put in the generator (in this case, it is  <code>picture</code>).**
 
```bash
 rails g paperclip item picture
```  
 After you are done, don't forget to migrate the database.:
```bash
 rails db:migrate
```   
 Go to the file of your model and add the following code:
 ```ruby 
 #app/models/item.rb
class Item < ApplicationRecord
  has_attached_file :picture, styles: { medium: "300x300>", thumb: "100x100>" }, default_url: "/images/:style/missing.png"
  validates_attachment :picture, presence: true
  do_not_validate_attachment_file_type :picture
end

```
Here is what each method is about:
 1. <code>has_attached_file</code> is the main method for adding a file attachment . The first argument is the attribute of the model that is going to be used for the file attachment (In this case it is<code>:picture</code>, as we know from the database migration). <code>styles:</code> is an **optional** parameter that is going to distribute the uploaded files in different folders according to their file size. In this example, medium photos will be 300x300 pixels and up and will be put into the <code>/images/medium</code> directory. <code> default_url </code> is also an **optional** parameter used to specify the path of a default image that will be returned if the object from the model does not have an attached image. You can put the default image in <code> app/assets/images/(medium or thumb)/</code> . If you are uploading files that are different from images, you may omit the optional arguments.

 2. <code>validates_attachmentt</code> is can validate the **content-type** , **presence** and **size** of the file. You can see how the syntax for all the validations [here](http://www.rubydoc.info/github/thoughtbot/paperclip/Paperclip%2FValidators%2FClassMethods%3Avalidates_attachment). In this example, only the presence is checked.
 
 3. <code> do_not_validate_attachment_file_type</code> is used as there are [issues](https://github.com/thoughtbot/paperclip/issues/1429) with the file type validations of paperclip.
  
The last step is to add the <code>:picture</code> to the jbuilder view, so that when we <code>GET</code> an item, it will return its picture:
```ruby   
json.extract! @item, :id, :name, :description, :picture, :created_at, :updated_at
```  

### Uploading multiple files

#### Adding document model
To make uploading multiple files using Paperclip possible, the most efficient way is to do so is by **adding another model that is going to  be related to the main model** and contain the files. In the current example, let's add the capability for an item to have multiple PDF documents. First, let's generate a model:

```bash
 rails g model document item:references
 rails g paperclip document file

```
The first generator is going to generate a model named <code>document</code> with a name as a string. <code>item:references</code> will generate a <code> belongs_to :item </code> reference which means that the item and the document will have a [one-to-many](http://www.databaseprimer.com/pages/relationship_1tox/) relationship. As it was already mentioned, second generator will generate fields for the attachment.

A reference also has to be added to the <code> Item </code> model as well:
```ruby  
class Item < ApplicationRecord
  #app/models/item.rb
  has_many :documents
  #...
end
```
Migrate the changes to the database schema:
```bash
 rails db:migrate
```
With this step finished, the <code> document </code> model is ready to be configured. Let's add the necessarry methods and validations to add PDF documents.
```ruby 
 #app/models/document.rb
 class Document < ApplicationRecord
  belongs_to :item
  has_attached_file :file
  validates_attachment :file, presence: true, content_type: { content_type: "application/pdf" }
end
```
This time, <code> validates attachment </code> checks if the document's type is <code> application/pdf </code>. 
The model is ready, here is how things have to be handled in the controller in order to handle creation of multiple files:
#### Modifying the item model
When a new item is created, there has to be another parameter sent, <code>document_data</code> which will contain an array of data about each document, either in multipart or in base64 format:

```ruby
    #app/contollers/items_controller.rb
    def item_params
      params.require(:item).permit(:name, :description , :document_data => []) #add :documents_data in permit() to accept an array 
    end
```


Add the logic for creating multiple files in the controller:



```ruby
  #app/controllers/items_controller.rb
   #...
  def create
    @item = Item.new(item_params)

    if @item.save
      @item.save_attachments(item_params) if params[:item][:document_data]
      render :show, status: :created, location: @item
    else
      render json: @item.errors, status: :unprocessable_entity
  end
    #...
  end
```
After the item is successfully saved, the <code> document_data </code> parameter has to be checked. If it is not empty, the controller will call the <code>save_attachments</code> that will attempt to create the documents. Here is how the model method has to look like:
```ruby
#app/models/item.rb
class Item < ApplicationRecord
  #...
  def save_attachments(params)
    params[:document_data].each do |doc|
      self.documents.create(:file => doc)
    end
  end
  #...
end
```
The <code> save_attachments </code> method is going to go through all the items in the <code>document_data</code> array.
For each document, there is going to be an object from the document model created.


All done! Now it is possible to add multiple files to a document.
Add <code>:documents</code> to the Juilder view so that the documents get returned together with the item.
```ruby   
 #app/views/items/show.json.jbuilder
json.extract! @item, :id, :name, :description, :picture, :documents, :created_at, :updated_at
```  

#### Testing it out
```bash
curl -F "item[document_data][]=@E:\file2.pdf;type=application/pdf"  -F "item[document_data][]=@E:\file1.pdf;type=application/pdf" -F   "item[picture]=@E:\photo.jpg" -F "item[name]=item" -F "item[description]=desc"  localhost:3000
/items
``
### base64 upload
To upload a base64-encoded image, the <code>attr_accessor</code> method will be used to get the base64 encoded string in the model. In your item model, add the following line:
 ```ruby
  #app/models/item.rb
class Item < ApplicationRecord
 attr_accessor :image_base
     #paperclip logic
end
```
Now, a method for decoding <code>image_base</code> and assigning it to the <code> picture </code> attribute of the Item.


 ```ruby 
  #app/models/item.rb
 class Item < ApplicationRecord
  #paperclip logic
  private

    def parse_image
    image = Paperclip.io_adapters.for(image_base)
    image.original_filename = "file.jpg"
    self.picture = image
  end

``` 
The <code>parse_image </code> method takes <code>image_base</code> and puts it into Paperclip's [IO adapters](http://www.rubydoc.info/gems/paperclip/Paperclip/AdapterRegistry#registered_handlers-instance_method) . They contain a registry which can decode the base64 string back to binary. Because the file name is not stored, you can either ut an arbitrary value (like "file.jpg", even if your file is not in jpg format) or add another <code>atr_accessor</code> for the name itsel. Finally <code> self.photo = image </code> assigns the image to the current instance of the object.

The method is ready, but it has to be called every time a new object is created, so let's add a filter that will call the <code> parse_image </code> method when that happens:
 
  ```ruby 
 #app/models/item.rb
class Item < ApplicationRecord
  before_validation :parse_image
  #...
end
```

The <code>before_validation</code> method will ensure that the base64 string will be parsed and assigned to the object instance before the validation i.e. before Paperclip validates the size, content type and the rest of the attributes of the file.
  
 Here is how the <code> Item </code> model should look with everything implemented:
```ruby 
   #app/models/item.rb
class Item < ApplicationRecord
    attr_accessor :image_base
    has_many :documents
    before_validation :parse_image
    
    has_attached_file :picture, styles: { medium: "300x300>", thumb: "100x100>" }, default_url: "/images/:style/missing.png"
    validates_attachment :picture, presence: true
    do_not_validate_attachment_file_type :picture
    
    private
    
        def parse_image
        image = Paperclip.io_adapters.for(image_base)
        image.original_filename = "file.jpg"
        self.picture = image
        end
end
```
 In order to get <code>image_base</code>, it has to be passed as a parameter. This means it has to be white-listed first:
```ruby 
    #app/controllers/items_controller.rb
    def item_params
      params.require(:item).permit(:name, :description , :documents, :image_base) #add :imnage_base in permit() 
    end
```

##  File upload using Carrierwave

