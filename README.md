# AWS S3 Cheatsheet for Flask

<br>

## **Getting Started:**

  - To get started using AWS S3 in your project, we need to install the following dependancy:  

  ```
  pipenv install boto3
  ```

<br>

  - Next make sure to set the following variables in your backend .env file:

  ```
  AWS_BUCKET_NAME=<BUCKET NAME>
  AWS_ACCESS_KEY=<USER ACCESS KEY>
  AWS_SECRET_ACCESS_KEY=<USER SECRET KEY>
  AWS_DOMAIN=http://<YOUR BUCKET NAME>.s3.amazonaws.com/
  ```

<br>

***

## **Setting up the API call:**

  - When sending the form data to your store or route:
    - Pull the data from the object.
    - Create a new variable using FormData().
    - Append all the data to the FormData() variable.
    - Remove the headers from the fetch call, we won’t need them.
    - Should look something like this:

<br>

  ```
  export const createPostThunk = (postObj) => async (dispatch) => {

    const {user_id, caption_content, image_url} = postObj;
    const formData = new FormData();

    formData.append("user_id", user_id)
    formData.append("caption_content", caption_content)
    formData.append("image_url", image_url)


    const response = await fetch('/api/posts/', {
        method: "POST",
        body: formData
    });

    if (response.ok) {
        const data = await response.json();
        dispatch(createPost(data))
        return null;
    }
    else if (response.status < 500) {
        const data = await response.json();
        return data
    }
    else {
        return { serverError: "An error occurred. Please try again." }
    }
  }
  ```

  > *Note: Variables may be different depending on your form.*

<br>

***

## **Creating the Upload Function:**

  - Somewhere on the backend create a file for our helper function.
  > *Example:*

<br>

  ```
  import boto3, botocore
  import os
  from werkzeug.utils import secure_filename

  s3 = boto3.client(
      "s3",
      aws_access_key_id=os.getenv('AWS_ACCESS_KEY'),
      aws_secret_access_key=os.getenv('AWS_SECRET_ACCESS_KEY')
  )

  def upload_file_to_s3(file, acl="public-read"):
      filename = secure_filename(file.filename)
      try:
          s3.upload_fileobj(
              file,
              os.getenv("AWS_BUCKET_NAME"),
              file.filename,
              ExtraArgs={
                  "ACL": acl,
                  "ContentType": file.content_type
              }
          )

      except Exception as e:
      # This is a catch all exception.
          print("Something Happened: ", e)
          return e


      # After uploading file to s3 bucket, return filename
      return file.filename
  ```
***

<br>

## **Using our Upload Function in the Post Route:**

  - Modify route to call the function to upload the file to aws before creating your new post.
  > *Example:*

<br>

  ```
  @post_routes.route('/', methods = ['POST'])
  @login_required
  def create_post():
      
      # Create a new post.
      
      post_form = PostForm()


      # inject csrf token
      post_form['csrf_token'].data = request.cookies['csrf_token']

      # Get our image from the request
      file = request.files["image_url"]

      # If we have a file, upload it to our bucket
      if file:
          # output will have the return value of the file name itself
          output = upload_file_to_s3(file)


      # validate form
          if post_form.validate_on_submit():
              # Create a new post
              new_post = Post(
                  user_id = post_form["user_id"].data,
                  caption = post_form["caption_content"].data,
                  image_url = f"http://<YOUR BUCKET NAME HERE>.s3.amazonaws.com/{output}"
              )
              db.session.add(new_post)
              db.session.commit()
              return new_post.to_dict()


      # return invalid errors
      return post_form.errors, 400
  ```

<br>

  > **Check to see if your file was uploaded to your bucket, this is a very simplistic example of how to get started and you will need to modify the code to your needs.**