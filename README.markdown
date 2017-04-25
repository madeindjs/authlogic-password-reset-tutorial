# Authlogic Password Reset Tutorial

## Introduction

This tutorial will help you set up password reset restfully, with
[Authlogic](http://github.com/binarylogic/authlogic).
This tutorial is based on
[this blog post](http://www.binarylogic.com/2008/11/16/tutorial-reset-passwords-with-authlogic/).
Hopefully, having the tutorial as a Git repository, it will be more up
to date.

To reset a password, the user goes through these steps:

1. The user requests a password reset
2. An email is sent to the user with instructions
3. The user verifies their identity by using the link in the email
4. The user is presented with a simple form for updating the password

This tutorial includes all code, including tests. Below you'll see the
tools that are used. If you're not familiar with any of them, check
them out or replace them with your favorite tools.

* [Shoulda](http://github.com/thoughtbot/shoulda)

_Note that this tutorial assumes you have your user and user session stuff already set up._


## Perishable Token

Perishable token as the name implies is a temporary string. Authlogic
use it for simple identification. Add a column called
**perishable_token** to your table and Authlogic will basically handle
the rest.

Generate the migration:

    $ rails generate migration add_perishable_token_to_users

The migration contents:

    class AddPerishableTokenToUsers < ActiveRecord::Migration[5.0]
      def change
        add_column :users, :perishable_token, :string, :default => "", :null => false
        add_index :users, :perishable_token
      end

      def down
        remove_column :users, :perishable_token
      end
    end

Don't forget to migrate the database:

    $ rake db:migrate


## Password Resets controller

Next up we add a controller called _password resets_:

    $ rails generate controller password_resets

With this contents:

    class PasswordResetsController < ApplicationController
      before_action :load_user_using_perishable_token, :only => [ :edit, :update ]

      def new
      end

      def create
        if @user = User.find_by_email(params[:email])
          @user.deliver_password_reset_instructions!
          flash[:notice] = "Instructions to reset your password have been emailed to you"
          redirect_to root_path
        else
          flash.now[:error] = "No user was found with email address #{params[:email]}"
          render :new
        end
      end

      def edit
      end

      def update
        @user.password = params[:password]
        # Only if your are using password confirmation
        # @user.password_confirmation = params[:password]

        # Use @user.save_without_session_maintenance instead if you
        # don't want the user to be signed in automatically.
        if @user.save
          flash[:success] = "Your password was successfully updated"
          redirect_to @user
        else
          render :edit
        end
      end


      private

      def load_user_using_perishable_token
        @user = User.find_using_perishable_token(params[:id])
        unless @user
          flash[:error] = "We're sorry, but we could not locate your account"
          redirect_to root_url
        end
      end
    end


Add this route:

    resources :password_resets, :only => [ :new, :create, :edit, :update ]

### Views

#### Action: new

    <h1>Reset Password</h1>
     
    <p>Please enter your email address below and then press "Reset Password".</p>
     
    <%= form_tag password_resets_path do %>
      <%= text_field_tag :email %>
      <%= submit_tag "Reset Password" %>
    <% end %>


#### Action: edit

    <h1>Update your password</h1>
     
    <p>Please enter the new password below and then press "Update Password".</p>
     
    <%= form_tag password_reset_path, :method => :put do %>
      <%= password_field_tag :password %>
      <%= submit_tag "Update Password" %>
    <% end %>


## Functional test

Create a user in fixture *test/fixtures/users.yml*

    me:
      id: 1
      email: madeindjs@gmail.com
      firstname: Alexandre 
      slug: alexandre
      lastname: Rousseau
      activated: true
      password_salt: <%= salt = Authlogic::Random.hex_token %>
      crypted_password: <%= Authlogic::CryptoProviders::Sha512.encrypt("20462046" + salt) %>
      persistence_token: <%= Authlogic::Random.hex_token %>
      perishable_token: <%= Authlogic::Random.hex_token %>

and now update your Unit Test like this

    class PasswordResetsControllerTest < ActionDispatch::IntegrationTest

      setup do
        @user = users(:me)
      end

      test "should get new" do
        get new_password_reset_url
        assert_response :success
      end

      test "should create " do
        post password_resets_url, params: {email: @user.email}
        assert_redirected_to root_url
      end

      test "should get edit" do
        get edit_password_reset_url
        assert_response :success
      end

      test "should update" do 
        put password_reset_url(@user.perishable_token), params: {user: { password: "newpassword" } }
        assert_redirected_to user_url(@user)
      end

    end


## The mail
As you might noticed, in the password resets controller there is a
method call on the user to **deliver_password_reset_instructions!**.

Lets add that method to the user model:

    class User < ApplicationRecord
      def deliver_password_reset_instructions!
        reset_perishable_token!
        UserMailer.deliver_password_reset(self).deliver_now
      end
    end

And test it:

    test "should delivering password instructions" do
      assert_empty @user.perishable_token
      @user.deliver_password_reset_instructions!
      assert_not_empty @user.perishable_token
    end

Add the mailer method:

    class UserMailer < ApplicationMailer
      def password_reset_instructions user
        @url = edit_password_reset_url(user.perishable_token)
        mail subject: "Password Reset Instructions", to: user.email
      end
    end

### Mailer view

    <h1>Password Reset Instructions</h1>
     
    <p>
      A request to reset your password has been made. If you did not make
      this request, simply ignore this email. If you did make this
      request, please follow the link below.
    </p>
     
    <%= link_to "Reset Password!", @url %>


### Test
Test the mailer:

    class UserMailerTest < ActionMailer::TestCase
      context "delivering password reset instructions" do
        setup do
          @user = Factory(:user)
          @user.deliver_password_reset_instructions!
        end
     
        should "send an email" do
          assert_sent_email do |email|
            email.subject =~ /Password Reset Instructions/
            email.body =~ /#{@user.perishable_token}/
            email.body =~ /Password Reset Instructions/
          end
        end
      end
    end

## Sum up

Feel free to contact me if there's anything in the tutorial that is
incorrect or if you have any good improvement suggestions.

If you liked this tutorial, I can also recommend the
[Authlogic Activation Tutorial](http://github.com/matthooks/authlogic-activation-tutorial).
