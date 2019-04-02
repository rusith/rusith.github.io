---
layout: post
title: Service Helper — Seamlessly Send Request in Angular
tags: programming angular typescript web
comments: true
---

When using Angular 2+ to build web stuff , we use services to get data from our back-end services. usually most of these service calls are HTTP requests. Angular provides the `Http` class to send HTTP request, this has built on the `Observable` API. When we send a request, it returns an Observable which can be used to use the response returned by the server.

Below is a Helper class which has built up on this Angular’s API. Which can be easily injected to a service and can be used to build HTTP requests seamlessly with its API that implements the Builder pattern and the Promise API

I have added token authorization support to this implementation. I have to set the `accessToken` property of the service helper instance whenever i change the user token in the app.

I have stored the base URL for my API in the `process.env.apiHost`, This could come from anywhere.

You can use multiple APIs by adding another function like the `api` function in the service helper.

```ts
import { Injectable, EventEmitter } from '@angular/core';
import { Http, RequestOptions, Headers, Response } from '@angular/http';
import { Observable } from 'rxjs/Rx';

declare const process: any;

/**
 * This is a helper class that will help you  to send Http Requests
 */
@Injectable()
export class ServiceHelper {
  public accessToken: string;
  public fetchStart = new EventEmitter<any>();
  public fetchEnd = new EventEmitter<any>();
  constructor(protected _http: Http) { }
  public api(): IRequestBuilder {
    return new RequestBuilder(this._http, process.env.apiHost, new UserTokenProvider(this), this.fetchStart, this.fetchEnd);
  }

}

export interface IRequestBuilder {
  url(url: string): IRequestBuilder;
  header(name: string, value: string): IRequestBuilder;
  noAuth(): IRequestBuilder;
  post<T>(): Promise<T>;
  put<T>(): Promise<T>;
  patch<T>(): Promise<T>;
  delete<T>(): Promise<T>;
  needJson(): IRequestBuilder;
  needUrlencode(): IRequestBuilder;
  hasJson(): IRequestBuilder;
  hasUrlencoded(): IRequestBuilder;
  rowData(data: any): IRequestBuilder;
  json(data: any): IRequestBuilder;
  get<T>(queyParameters?: string): Promise<T>;
  getObserve<T>(queyParameters?: string): Observable<T>;
  sendFile(files: File[], timenow: any, progress: any);
}

interface AuthTokenProvider {
  getTokenInfo(): { name: string, value: string };
}

export type NextCallback<T> = (value: T) => void;
export type ErrorCallback<T> = (value: T) => void;


class UserTokenProvider implements AuthTokenProvider {
  private _serviceHelper: ServiceHelper;
  constructor(serviceHelper: ServiceHelper) { this._serviceHelper = serviceHelper; }
  getTokenInfo(): { name: string, value: string } {
    return this._serviceHelper.accessToken ? { name: 'Authorization', value: ` bearer ${this._serviceHelper.accessToken}` } : null;
  }
}

export class RequestBuilder implements IRequestBuilder {

  private _helper: ServiceHelper;
  private _request: {
    url: string, headers: Array<{ name: string, value: string }>,
    authorize: boolean, data: string
  } = { url: '', headers: [], authorize: true, data: '' };
  private _http: Http;
  private _baseUrl: string;
  private _tokenProvider: AuthTokenProvider;
  private _startEvent: EventEmitter<any>;
  private _endEvent: EventEmitter<any>;

  constructor(http: Http, baseUrl: string, tokenProvider: AuthTokenProvider, start: EventEmitter<any>, end: EventEmitter<any>) {
    this._http = http;
    this._baseUrl = baseUrl;
    this._tokenProvider = tokenProvider;
    this._startEvent = start;
    this._endEvent = end;
    this._request.authorize = true;
  }

  private getRequestOptions(): RequestOptions {
    const r = this._request;
    const requestOptions = new RequestOptions();
    if (r.authorize) {
      const tokenInfo = this._tokenProvider.getTokenInfo();
      if (tokenInfo) {
        r.headers.push({ name: tokenInfo.name, value: tokenInfo.value });
      }
    }
    if (r.headers && r.headers.length) {
      const headers = new Headers();
      r.headers.forEach(header => {
        headers.append(header.name, header.value);
      });
      requestOptions.headers = headers;
    }
    requestOptions.withCredentials = false;
    return requestOptions;
  }

  url(url: string): IRequestBuilder {
    this._request.url = url;
    return this;
  }

  rowData(data: any): IRequestBuilder {
    this._request.data = data;
    return this;
  }

  json(data: any): IRequestBuilder {
    this._request.data = JSON.stringify(data);
    return this.hasJson();
  }

  header(name: string, value: string): IRequestBuilder {
    this._request.headers.push({ name, value });
    return this;
  }

  noAuth(): IRequestBuilder {
    this._request.authorize = false;
    return this;
  }

  needJson(): IRequestBuilder {
    this._request.headers.push({ name: 'Accept', value: 'application/json' });
    return this;
  }

  needUrlencode(): IRequestBuilder {
    this._request.headers.push({ name: 'Accept', value: 'application/x-www-form-urlencoded' });
    return this;
  }

  hasJson(): IRequestBuilder {
    this._request.headers.push({ name: 'Content-Type', value: 'application/json' });
    return this;
  }

  hasUrlencoded(): IRequestBuilder {
    this._request.headers.push({ name: 'Content-Type', value: 'application/x-www-form-urlencoded' });
    return this;
  }

  post<T>(): Promise<T> {
    return new Promise((resolve, reject) => {
      this._startEvent.emit();
      const subscription = this._http.post(this._baseUrl + this._request.url, this._request.data, this.getRequestOptions())
        .subscribe((response) => { this.next(resolve, response); this._endEvent.emit(); },
          (e) => { subscription.unsubscribe(); this._endEvent.emit(e); this.error(e, reject); });
    });
  }


  put<T>(): Promise<T> {
    return new Promise((resolve, reject) => {
      this._startEvent.emit();
      const subscription = this._http.put(this._baseUrl + this._request.url, this._request.data, this.getRequestOptions())
        .subscribe((response) => { this.next(resolve, response); this._endEvent.emit(); },
          (e) => { subscription.unsubscribe(); this._endEvent.emit(); this.error(e, reject); });
    });

  }

  patch<T>(): Promise<T> {
    return new Promise((resolve, reject) => {
      this._startEvent.emit();
      const subscription = this._http.patch(this._baseUrl + this._request.url, this._request.data, this.getRequestOptions())
        .subscribe((response) => { this.next(resolve, response); this._endEvent.emit(); },
          (e) => { subscription.unsubscribe(); this._endEvent.emit(); this.error(e, reject); });
    });

  }

  delete<T>(): Promise<T> {
    return new Promise((resolve, reject) => {
      this._startEvent.emit();
      const subscription = this._http.delete(this._baseUrl + this._request.url, this.getRequestOptions())
        .subscribe((response) => { this.next(resolve, response); this._endEvent.emit(); },
          (e) => { subscription.unsubscribe(); this._endEvent.emit(); this.error(e, reject); });
    });

  }

  get<T>(queyParameters = ''): Promise<T> {
    return new Promise((resolve, reject) => {
      this._startEvent.emit();
      const subscription = this._http.get(this._baseUrl + this._request.url + '?' + queyParameters, this.getRequestOptions())
        .subscribe((response) => { this.next(resolve, response); this._endEvent.emit(); },
          (e) => { subscription.unsubscribe(); this._endEvent.emit(); this.error(e, reject); });
    });
  }

  getObserve<T>(queyParameters = ''): Observable<T> {
    this._startEvent.emit();
    return this._http.get(this._baseUrl + this._request.url + '?' + queyParameters, this.getRequestOptions())
      .map(r => this.next(null, r));
  }

  sendFile(files: File[], timenow: any, progress: any): Observable<any> {
    const url = this._baseUrl + this._request.url;
    return Observable.create(observer => {
      const formData: FormData = new FormData(),
        xhr: XMLHttpRequest = new XMLHttpRequest();

      for (let i = 0; i < files.length; i++) {
        formData.append('uploads[]', files[i], files[i].name);
      }

      xhr.onreadystatechange = () => {
        if (xhr.readyState === 4) {
          if (xhr.status === 200) {
            observer.next(JSON.parse(xhr.response));
            observer.complete();
          } else {
            observer.error(xhr.response);
          }
        }
      };
      xhr.upload.onprogress = (event) => {
        progress(Math.round(event.loaded / event.total * 100));
      };
      xhr.open('POST', url, true);

      if (this._request.authorize) {
        const tokenInfo = this._tokenProvider.getTokenInfo();
        if (tokenInfo) {
          xhr.setRequestHeader(tokenInfo.name, tokenInfo.value);
        }
      }
      const serverFileName = xhr.send(formData);
    });
  }

  private error(e, callback) {
    if (e.status === 404) {
      callback('Requested resource not found');
    } else {
      callback(e.json());
    }
  }

  private next(n, data): any {
    try {
      const d = data.json();
      if (n) {
        n(d);
      }
      return d;
    } catch (e) {
      if (n) {
        n({});
      }
      return {};
    }
  }
}
```

## How to Use

Below is a very simple usage of the service helper class. Which has some calls to an order API

When using the service, you can handle errors just like you do with normal promises.

```ts
import { Injectable } from '@angular/core';
import { ServiceHelper } from '../base/service.helper';
import OrderModel from './../models/order.model.ts';

@Injectable()
export default class OrderService {

  /* We inject the service helper */
  constructor(private _: ServiceHelper) {}

  getAllOrders(searchText = ''): Promise<Array<OrderModel>> {
    return this._.api()
      .url('orders')
      .needJson()
      .get(`searchText=${searchText}`);
  }

  getAnOrder(id: number): Promise<Array<OrderModel>> {
    return this._.api()
      .url(`orders/${id}`)
      .needJson()
      .get();
  }

  addAnOrder(order: OrderModel): Promise<OrderModel> {
    return this._.api()
      .url('orders')
      .json(order)
      .needJson()
      .post();
  }

  updateAnOrder(order: OrderModel): Promise<OrderModel> {
    return this._.api()
      .url(`orders/${order.id}`)
      .json(order)
      .needJson()
      .patch();
  }

  deleteAnOrder(id: number): Promise<OrderModel> {
    return this._.api()
      .url(`orders/${id}`)
      .needJson()
      .delete();
  }
}
```
